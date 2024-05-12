描述 etcd 中的 watch 机制。

几个问题：
- 区间树是个啥 ？

# watch 有什么用

一个典型的场景是 leader 选举，follower 如何及时感知 leader  下线，进而发起 leader 选举呢？<br>

没有 watch 机制之前，我们只能靠轮询的方式判断 leader 是否下线。有了 watch 机制之后，leader 可以在 etcd 注册一个跟 lease 绑定的 key，然后定期刷新该 lease，然后集群中的 follower watch 这个 key，当 leader 下线后，key 到期后会被删除，这样其他 follower 就能快速感知到 leader 下线，进而发起 leader 选举。

# watch 原理

- grpc stream

    watch 底层实现是基于 grpc http2 stream 的特性实现的，当客户端通过 stream 发送一个 watch 请求后，当服务端检测到对应的事件后，会通过 stream 不断的给客户端推送事件。

- watcher 分类

    watcher 实现的大体逻辑是：当 TxnWrite 有变更在提交的时候，会通过 key 匹配对应的 watcher，然后基于 key/value pair 构建对应的 event，然后将 watcher/events pair 发送到一个 ch，服务端入口负责从该 ch 读取事件，然后通过 stream 发送给客户端。<br>

    但这里需要注意一个问题，我们要保证一个 watcher 接收到的 events 是按照 revision 排序的。所以，如果某个 watcher 当前检测的 min_revision <= store.cur_revision，那意味这在 [min_revision, store.cur_revision] 还是可能存在我们感兴趣的事件的。因此在 TxnWrite commit 的时候，不能直接将对应的 events 发送给 ch。<br>

    因此会将 watcher 分为两类：synced_watcher、unsynced_watcher。etcd 会在后台拉起一个异步协程，负责给 unsynced_watcher 发送 events，当追赶上最新的 revision 的时候，就会被加入到 synced_watcher。<br>

    但这里还有个问题，所有 watcher/events pair 是发送到一个 ch 的，这个 ch 大小是有限制的，如果 ch 满了，怎么办？本次 watcher/events pair 发送就会阻塞。为了不保证阻塞，etcd 将此类 watcher 标价为 victim(受害者)，同样在异步协程中会尝试将此 watcher/events pair 发送到 ch，如果发送成功了，在根据是否追赶上最新 revision，将 watcher 发送到 synced_watcher 或者 unsynced_watcher 中。

- client 重试
  
    watcher 并不会在服务端持久化，当节点挂掉后，所有 watcher 相关信息就会丢失。etcd 的解决方式时：会记录最近一次接收到 event 的 revision，然后基于此 revision 向其他 etcd 节点发起一次 watch 请求。<br>

    这个给我们的思路是：并非所有的逻辑都需要在服务端做，也可以尝试在 client sdk 中加入一些逻辑来实现一些功能。

# todo

- 了解下区间树的实现原理
- 之前了解了 grpc stream，但还是不太清楚，如何使用，etcd watch 是一个典型的例子，有机会可以看看。