# client 如何感知 etcd 集群的变化

我们知道 etcd 集群实例信息是会发生变化的，比如新增/下线实例。client 也需要感知这些信息，进而才能将流量均衡的打到 etcd cluster 上。<br>
那么 client 是如何感知这些信息的呢？<br>

实际上，在创建 client 的时候，我们可以指定拉起一个后端协程，该协程定时访问集群获取集群中所有成员，然后更新到 client 对应的 resolver 中。<br>
我们来看看这部分的代码实现：

```go
func newClient(cfg *Config) (*Client, error) {
	ctx, cancel := context.WithCancel(baseCtx)
	client := &Client{
		conn:     nil,
		cfg:      *cfg,
		creds:    creds,
		ctx:      ctx,
		cancel:   cancel,
		mu:       new(sync.RWMutex),
		callOpts: defaultCallOpts,
		lgMu:     new(sync.RWMutex),
	}

	// Use a provided endpoint target so that for https:// without any tls config given, then
	// grpc will assume the certificate server name is the endpoint host.
	conn, err := client.dialWithBalancer()
	if err != nil {
		client.cancel()
		client.resolver.Close()
		// TODO: Error like `fmt.Errorf(dialing [%s] failed: %v, strings.Join(cfg.Endpoints, ";"), err)` would help with debugging a lot.
		return nil, err
	}
	client.conn = conn

    // 该协程 sync 的就是 cluster member 信息
	go client.autoSync()
	return client, nil
}
```

接着，我们看看 autoSync() 的实现：
```go
func (c *Client) autoSync() {
	// 如果没设置此选项，标识不进行 sync，直接退出。
	if c.cfg.AutoSyncInterval == time.Duration(0) {
		return
	}

	for {
		select {
		case <-c.ctx.Done():
			return
		case <-time.After(c.cfg.AutoSyncInterval):
			ctx, cancel := context.WithTimeout(c.ctx, 5*time.Second)
			// 以固定的时间间隔获取集群中的 members 信息
			err := c.Sync(ctx)
			cancel()
			if err != nil && err != c.ctx.Err() {
				c.lg.Info("Auto sync endpoints failed.", zap.Error(err))
			}
		}
	}
}
```

```go
func (c *Client) Sync(ctx context.Context) error {
	// 获取集群中的 member 信息
	mresp, err := c.MemberList(ctx)
	if err != nil {
		return err
	}
	var eps []string
	for _, m := range mresp.Members {
		if len(m.Name) != 0 && !m.IsLearner {
			eps = append(eps, m.ClientURLs...)
		}
	}
	// 设置 endpoints，本质就是设置 grpc clientConnection 对应的 resolver
	c.SetEndpoints(eps...)
	return nil
}
```

实际上，客户端基本都是通过轮询的方式来感知集群信息变更的。比如在 kafka 中，producer 要分区，就需要知道集群的元信息，就是通过轮询的方式获取集群元信息的。<br>
再比如 rocketmq 中，producer 要发送消息到某个 topic，怎么知道 topic 对应的 broker 呢?其实是从 nameserver 获取的，但是 nameserver 节点也可能<br>
是变化的，producer 也是通过定时轮询某个 nameserver 来获取当前所有的 nameserver 实例，进而能够及时感知 nameserver 的变动，避免将请求发送到<br>
已经不存在的 nameserver 实例上。