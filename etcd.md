描述 etcd 相关的内容。


# 使用

# 项目架构

# server

## mvcc

## 事务

- 事务的执行顺序？并行/串行？

- 一个事务是不是对应一个 log?

- 事务原子性如何保证？

- 事务隔离性如何实现？

## boltdb

- node 什么用途？

    事务提交前的写都是写入到 node 了吗？

- 都包括哪些锁？粒度是什么样的？

## buffer

- 存在的目的是什么？读事务为什么不能直接从 boltdb 读？

# client

## etcdctl

- 可以看看一个命令行工具怎么写，针对不同的子命令，是如何实现的？

开源库：github.com/spf13/cobra

## 如何测试

raft 是怎么测试的呢？


# 使用

## 使用场景

- 分布式协调服务
- 存储配置
- 服务发现
- 分布式锁

# 观测性

- 如何排错/定位 ？

# 网络

- 如何即支持 http 又支持 grpc 的 ？

# 其他

- 如果某些节点 apply 失败，一些节点 apply 成功，会发生什么