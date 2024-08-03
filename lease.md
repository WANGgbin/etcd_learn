几个问题：
- 什么是 lease ?
- 有什么用，使用常见都包括啥？
- 怎么实现的 ？

# lease

lease 即租约的意思，key 可以跟一个 lease 绑定，当 lease 过期的时候，就会自动删除 lease 以及关联的所有的 key。

lease 常见的操作包括：
- 创建
- 续期
- 删除
- key attach lease
- key unattach lease


# 使用场景

- leader 选举

一些分布式服务可以基于 etcd 来实现，其中有一个需求是只保证任何时候有且只有一个 leader，怎么实现呢？<br>

leader 节点可以在 etcd 中创建一个 key 并关联一个 lease，然后定时通过续约 lease 的方式来上报自己的健康状态。集群中的其他节点可以 watch 这个 key。一旦 leader 超过一定的时间都没有刷新 lease，那么 lease 以及关联的所有 key 都会被删除。这样集群中其他的节点就能及时收到 leader 下线的消息，进而发起新一轮选举。

- 分布式锁

另一个典型的场景就是分布式锁。可以让 key 绑定一个 lease，当 lease 过期的时候，其他节点才可以加锁。

# 实现原理

lease 只由 leader 负责管理，包括：创建、续约、删除等。follower 只负责从 leader 同步状态。<br>

leader 在内存中创建了 leaseID -> lease 的映射，同时每一个 lease 中包括所有关联的 key。同时在 server 启动的时候，会拉起一个异步的 goroutine，该 goroutine 是个定时任务，负责两件事：

- 根据 lease 的 expire 维护一个小根堆，定时检查此小根堆，监控到过期的 lease 的时候，通过 expiredC 发送消息给 server 主循环，server 会将此消息同步给所有的 follower，并删除对应的 lease 和 关联的 keys

- 如果 leader 挂了，当 follower 当选为 leader 后，需要重建对应的数据结构，因此 follower 也就需要知道每个 lease 对应的过期时间，**但是，不同的机器很可能时钟是不相同的，因此节点之间并不会直接同步绝对时间，而是同步时间偏移量，
在这里就是 remaining TTL。**所以，另一个任务就是定时更新 lease 的 remaining ttl， 并同步给所有的 follower。这里需要注意的是：如果更新 lease 的 remaining ttl 的时间间隔很小，导致的问题是整个 etcd 集群的流量会变多，影响整体的性能。
但如果更新频率低的话，就会导致 follower 维护的 remaining ttl 跟 实际的 remaining ttl 相差较多，这样当 follower 当选为 leader 的时候，该 lease remaining ttl 就不准确。目前这个间隔设计的默认值是 5min。

## 限流实现

如果某一时刻有大量的 lease 到期，在 leader 上删除大量的 lease，势必影响正常的业务处理。同时，也会在集群内部发送大量的 delete lease raft entry。所以，我们需要限制删除 lease 的 qps。那么 etcd 是怎么做的呢？<br>

etcd 通过两个步骤，限制了删除 lease 的 qps.<br>

- 每隔 500ms 从 heap 只取数量有限的到期 lease，具体上限取决于限制的 qps，比如限制 qps 为 10，这里 lease 的上限就是 5.
 
    有了第一步，还不够，因为完全有可能一下次就处理完这些到期的 lease，这样整体上看 1s 内，确实处理了 10 个 lease，但在这 1s 内，lease 处理不均衡。因此还有第二步。

- 第二部通过一个特定大小的 channel 限制处理从上一步获取到的 lease 的并发度。
    
    具体代码参考如下：
```go
	c := make(chan struct{}, maxPendingRevokes)
		for _, curLease := range leases {
			select {
			// 如果管道满了，表示达到了最大并发度，阻塞
			case c <- struct{}{}:
			case <-s.stopping:
				return
			}
            
            // 否则，就拉起一个 goroutine，处理当前 lease
			f := func(lid int64) {
				s.GoAttach(func() {
					ctx := s.authStore.WithRoot(s.ctx)
					_, lerr := s.LeaseRevoke(ctx, &pb.LeaseRevokeRequest{ID: lid})
					if lerr == nil {
						leaseExpired.Inc()
					} else {
						lg.Warn(
							"failed to revoke lease",
							zap.String("lease-id", fmt.Sprintf("%016x", lid)),
							zap.Error(lerr),
						)
					}

					<-c
				})
			}

			f(int64(curLease.ID))
		}
```

## 节点之间如何同步 lease 的 remaining ttl

etcd 是如何维护 lease 的 remaining ttl 的呢？我们知道 etcd 会持久化 lease 信息，这样如果节点 crash 了，重启后也是可以重建 lease 相关内容的，
但是我们也需要定时刷新 db 中 lease 的 remaining ttl，那么 etcd 是如何做的呢？<br>

一种简单的方式就是定时遍历所有的 lease，将 lease 的 remaining 刷新到 db 中。但这有两个问题：
- 效率差

  遍历所有的 lease，效率差

- 影响正常的业务请求处理

  因为有写 db 操作，如果对全部的 lease 都执行更新 remaining ttl 操作，势必影响正常的读写请求。

所以，etcd 通过以下两种措施来完成 remaining refresh。

- 限流

类似删除 lease 的限流。

- lease 设置 checkpoint

每个 lease 设置了一个表示 checkpoint 的 time，表示下次 check 的 时间点。然后将 lease 根据 checkpoint time 放在一个最小堆中，这样定时任务
每次遍历的时候，只需要返回需要 check 的 lease，而不是对所有的 lease 都 check。<br>

那么什么时候更新 checkpoint time 呢？当 lease 成功执行 checkpoint 即将 remaining ttl 刷新到 db 后，会再次更新 lease 的 checkpoint time.
