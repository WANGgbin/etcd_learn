描述 etcd 中的 watch 机制。

几个问题：
- 区间树是个啥 ？

# watch 有什么用

一个典型的场景是 leader 选举，follower 如何及时感知 leader  下线，进而发起 leader 选举呢？<br>

没有 watch 机制之前，我们只能靠轮询的方式判断 leader 是否下线。有了 watch 机制之后，leader 可以在 etcd 注册一个跟 lease 绑定的 key，然后定期刷新该 lease，然后集群中的 follower watch 这个 key，当 leader 下线后，key 到期后会被删除，这样其他 follower 就能快速感知到 leader 下线，进而发起 leader 选举。

# watch 原理

etcd 中的 watch 是通过一个 watch service 实现的，该 watch service 只有一个 Watch 方法，该方法是一个 req & resp 都是 stream 的 stream API。通过此 API，client 可以创建/取消 Watch.

- grpc stream

    watch 底层实现是基于 grpc stream 的特性实现的，当客户端通过 stream 发送一个 watch 请求后，服务端会创建两个协程，分别执行 recvLoop 和 sendLoop，recvLoop 用来处理 client stream，不断接受 client stream 上的 request，
sendLoop 用来处理 server stream，不断的发送 Watch Response 给客户端。<br>

- watch store

store 即 etcd 的 indexTree + boltdb 这一套。为了实现 watch 的特性，etcd 在 store 基础上又封装了一个 watch store 类型的 store。该 store 会在 storeWriteTxn 事务 End 的时候，给对应的 watcher 发送 events。<br>

watch store 的定义如下：

```go
type watchableStore struct {
	*store

	// mu protects watcher groups and batches. It should never be locked
	// before locking store.mu to avoid deadlock.
	mu sync.RWMutex

	// victims are watcher batches that were blocked on the watch channel
	victims []watcherBatch
	victimc chan struct{}

	// contains all unsynced watchers that needs to sync with events that have happened
	unsynced watcherGroup

	// contains all synced watchers that are in sync with the progress of the store.
	// The key of the map is the key that the watcher watches on.
	synced watcherGroup

	stopc chan struct{}
	wg    sync.WaitGroup
}
```

从上面的定义可以看到，etcd 创建的所有 watcher 都维护在 watchableStore 中。但需要注意的是，这些 watcher 并不是简单的组织在一个 map 中，而是分为若干个 map，包括：unsynced、synced、victims 这些对象的含义，我们后面介绍。<br>


storeTxnWrite 也对应了一个 watchableStoreTxnWrite，其 End() 实现如下：
```go
func (tw *watchableStoreTxnWrite) End() {
	changes := tw.Changes()
	// 如果没有变更直接调用底层 TxnWrite.End()
	if len(changes) == 0 {
		tw.TxnWrite.End()
		return
	}
    
    // 如果有变更，就需要将变更转化为 Events
	rev := tw.Rev() + 1
	evs := make([]mvccpb.Event, len(changes))
	for i, change := range changes {
		evs[i].Kv = &changes[i]
		if change.CreateRevision == 0 {
			evs[i].Type = mvccpb.DELETE
			evs[i].Kv.ModRevision = rev
		} else {
			evs[i].Type = mvccpb.PUT
		}
	}

	// end write txn under watchable store lock so the updates are visible
	// when asynchronous event posting checks the current store revision
	tw.s.mu.Lock()
	// 将 Events 通过给对应的 Watcher
	tw.s.notify(rev, evs)
	// 再调用 storeTxnWrite 的 End 方法，完成事务的提交
	tw.TxnWrite.End()
	tw.s.mu.Unlock()
}
```

- watcher 概念

用户每发起一个创建 watch 的请求，在 etcd server 内部，就会创建一个 watcher。该 watcher 定义如下：
```go
type watcher struct {
	// the watcher key
	key []byte
	// end indicates the end of the range to watch.
	// If end is set, the watcher is on a range.
	// 如果 end == nil，表示只 watch key 的变更，如果 end 不为空，表示 watch [key, end) 整个区间的变化。
    // 什么时候会监控一个区间的变化呢？ 一个场景是监控特定前缀的 key 的事件，
	end []byte

	// minRev is the minimum revision update the watcher will accept
	// 每个 watcher 会追踪一个 revision，只有大于该 revision 的 events，才会发送给 watcher，这么做的目的是为了避免重复发送事件给 watche
    // 同时 minRev 这个字段也是动态变化的。
	minRev int64
	
	// 每个 watcher 对应一个 id，后续用户取消 watcher，就是通过该 id 实现的
	id     WatchID

	// a chan to send out the watch response.
	// watch stream 对应的 channel
	ch chan<- WatchResponse
}
```

- watcher 管理 

watcher 的管理并不是一个简单的 map。这是为什么呢？<br>

是因为：**对于每一个 watcher，必须要按照 revision 的顺序发送 events 给 client，当一个写事务结束的时候，事务涉及的变更并不一定就是 watcher 关注的下一个变更，可能 watcher 注册的 revision 比较早。**

为了保证给 watcher 发送有序的 events，watchableStore 将 watcher 分为了三类：

 - synced watcher
  
  意味这该 watcher 的 minRev 已经追赶上了 store 的 current revision。这样每个写事务结束时，直接将对应的 events 发送给 watcher 即可。

 - unsynced watcher
  
  意味这该 watcher 的 minRev 还没有追赶上 store 的 current revision。每个写事务结束时，对应的变更不能直接发送 events 给 watcher。那怎么给这些 unsynced watchers 发送 events 呢？<br>

  有一个后台协程，会从 unsynced watcher 中跳出一批 watcher，然后根据每个 watcher 的 minRev 计算一个 minRev，然后创建一个 boltdb 的 readTxn，读取 [minRev, current revision] 之间所有的数据，然后将对应的 events
  发送给 watcher，当某个 watcher 的 minRev 追赶上 current revision 后，该 watcher 将变为 synced watcher.

 - victim
  
  每个 grpc watch stream 在 etcd server 中对应一个 watch stream，一个 watch stream 对应一个 channel。etcd 会将该 watch stream 中的 watcher 的事件通过该 channel 通知 api 层。<br>

  这里有个问题是，channel 大小是有限制的，由于生成/消费的速率差异，给 channel 发送 events 的时候，是有可能阻塞的，那怎么办？<br>

  etcd 的思路是，将这些 events 缓存到一个称为 victim 的对象中，同时为了保证有序性，对应的 watcher 要从 synced/unsynecd group 中删除。然后会有一个后台协程，定时尝试将 events 发送到对应的 channel。如果发送成功，
再将 watcher 发送到 synced/unsynced group 中。

- client 重试
  
    watcher 并不会在服务端持久化，当节点挂掉后，所有 watcher 相关信息就会丢失。etcd 的解决方式时：会记录最近一次接收到 event 的 revision，然后基于此 revision 向其他 etcd 节点发起一次 watch 请求。<br>

    这个给我们的思路是：并非所有的逻辑都需要在服务端做，也可以尝试在 client sdk 中加入一些逻辑来实现一些功能。

# todo

- 了解下区间树的实现原理
  