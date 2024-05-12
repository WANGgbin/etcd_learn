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

- 如果 leader 挂了，当 follower 当选为 leader 后，需要重建对应的数据结构，因此 follower 也就需要知道每个 lease 对应的过期时间，**但是，不同的机器很可能时钟是不相同的，因此节点之间并不会直接同步绝对时间，而是同步时间偏移量，在这里就是 remaing TTL。**所以，另一个任务就是定时更新 lease 的 remaing ttl，并同步给所有的 follower。这里需要注意的是：如果更新 lease 的 remaing ttl 的时间间隔很小，导致的问题是整个 etcd 集群的流量会变多，影响整体的性能。但如果更新频率低的话，就会导致 follower 维护的 reming ttl 跟 实际的 remaing ttl 相差较多，这样当 follower 当选为 leader 的时候，该 lease remaing ttl 就不准确。目前这个间隔设计的默认值是 5min。