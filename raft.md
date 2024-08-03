描述 etcd 中 raft 实现的一些细节。

## leader 选举

- prevote


## 日志同步

- 什么时候提交日志？

集群多数节点就该日志的全序(term + index)达成一致。但提交的日志可能是在 unstable storage 的，如果节点异常退出，丢失了怎么办 ？

- 提交的日志一定保证执行成功吗？如果是，如何实现的呢？
- 是否还会发生不一致问题？
- 日志需要持久化吗？为什么？如何做？
- 日志冲突怎么办 ？
- 从 WAL 中恢复的流程
- 复制状态转移图


## 如何广播

## 节点变更

### 添加新节点

- 新节点的数据是如何赶上 leader 的？

### 删除节点

## 快照

- 作用
- 如何实现？
- 什么时候创建快照

## WAL

- 每次写是不是都需要 fsync ? 如何提高性能呢 ？

## raft http

- peer 如何理解 ？
- 

## 其他

- 如何与应用层交互
- leaner 是什么 ？
- 心跳如何实现 ？ 
- 如何避免消息太多 ？
- follower 都有哪些复制状态 ？
- softState 和 hardState 是啥意思 ？
- cluster 信息是如何管理的 ？

# 线性读

## 什么是线性读

我们知道 etcd 是强(线性)一致性的。当一个写成功后，下一个读一定能够读到对应的数据。但是，我们知道写成功仅仅是 leader apply 成功，在其他 follower 节点上不一定 apply。这样在其他 follower 直接读不一定能够读到数据，那怎么办呢？<br>

这就是线性读要解决的问题，线性读的保证是即使在 follower 读取，也可以读取到最新数据。

## 实现原理

- 思路

    当 follower 获取到读请求的时候，会从 leader 获取 commit_index，然后会一直阻塞，直到 follower applied_index >= commit_index，才返回。

- 实现

    etcd 线性读的实现很巧妙。会拉起一个后台协程，该协程要做的就是，每一轮新建一个 ch，然后从 leader 获取 commited_index，然后使用 etcd 的 waitTime 机制阻塞，直到本地 applied_index >= commited_index，最后关闭本轮的 ch。<br>

    而请求端首先获取 ch，然后再 ch 阻塞，直到 ch 被关闭。

# 成员管理

# 全序关系广播

etcd 基于 raft 实现强一致性，raft 是**容错的共识算法**。所谓容错，是指可以容忍集群中若干节点的 crash，共识算法指的是就某一点能在集群中达成共识。<br>

而全序关系广播是实现共识的一种思路，而**日志**是实现全序关系广播的一种具体实现。etcd raft 就是通过 log 实现共识的。<br>

- 整体架构

我们以一个写流程来串联下日志同步的整个流程。注意：**并不是所有请求都需要过 raft 共识层，比如读请求**。<br>

    - 客户端发送写请求
    - 服务端收到后，将请求封装为一个 log entry，扔给 raft 层
    - raft 给此 log entry 分配一个 term 以及 index，注意此时 entry 尚未持久化，还在 unstable storage(内存) 中
    - 通过 raft 层提供的 Ready 接口，将分配好 term 和 index 的 entry 返回给 app
    - 因为 etcd raft 没有实现网络层、持久层，这些是在 etcd 的 app 层实现的，etcd 在 raft 层往上又封装了一个完整的 raft(包括持久层(wal)、网络层(raft_http)), etcd_raft 通过 raft 的 Ready 接口获取到 entry，然后进行持久化，同时发送 entries 给各个 follower。
    - follower 接收到 entries 后，会扔到自己的 unstable storage 中。然后发送 MsgAppResp 给 leader。
    - leader 收到响应后，根据所有 follower 的 match 信息更新 comitted_index，然后通过 Ready 接口将已经 commited_index 但是尚未 applied_index 的 entries 发送给 etcd_raft
    - etcd_raft 将 entries 通过 applyC ch 发送给 server
    - server 结束 entries，然后根据 consistent_index 实现幂等，获取已经 apply 的 entry，然后 apply 其他 entry
    - 当 entry apply 后，会给写请求对应的 ch 发送一个响应，服务端入口从 ch 接受这个响应，然后返回给客户端。

## unstable log

为什么要有 unstable log 呢？当接收到一个 entry 的时候，为什么不直接存到 WAL 中呢？实际上是为了**性能考虑**。不管是 client request 持久化到 leader 或者是 leader 广播到 follower，如果都是持久化 WAL，带来的必然是性能问题。<br>

所以提供了一个内存 buf: unstable log。entry 首先存储到 unstable log，然后再批量持久化到 WAL，这样可以提高系统整体性能。<br>

既然是内存 buf，那就有丢失的可能。如果 server crash 重启了怎么办？其实也没啥问题，因为集群中的其他节点保存了这些 entry。如果是 leader crash，则重启后，从新的 leader 同步 entry，如果是 follower 重启，直接从 leader 节点同步 entry.<br>

但如果是单节点的集群呢？如果仍然还按照上述逻辑，就会导致数据的丢失。因此，此场景下，在 apply entry 之前，一定会把 entry 持久化到 WAL 中，这样即使节点 crash，也可以从 WAL 中恢复日志。

## WAL

### 什么时候删除 WAL 中的日志

实际上当 wal 中的 entry applied 以后就没必要存储了，因为这些 entry 对应的变更已经存储到 db 了。 etcd 默认当 applied_index - sanp_index > 10000 的时候，就会生成快照 <br>

那么如何生成快照呢？我们直接看代码：
```go
func (s *EtcdServer) snapshot(snapi uint64, confState raftpb.ConfState) {
	clone := s.v2store.Clone()
	// commit kv to write metadata (for example: consistent index) to disk.
	//
	// This guarantees that Backend's consistent_index is >= index of last snapshot.
	//
	// KV().commit() updates the consistent index in backend.
    // 目的是持久化 consistent index
	s.KV().Commit()

    // 异步拉起 snapshot 操作
    // 以下几个操作必须是原子性的，我们看到 etcd 保证原子性的方式就是在出错的时候 panic
	s.GoAttach(func() {
		lg := s.Logger()

		// raft 稳定存储生成 snap
		snap, err := s.r.raftStorage.CreateSnapshot(snapi, &confState, d)
		if err != nil {
			// the snapshot was done asynchronously with the progress of raft.
			// raft might have already got a newer snapshot.
			if err == raft.ErrSnapOutOfDate {
				return
			}
			lg.Panic("failed to create snapshot", zap.Error(err))
		}
		// SaveSnap saves the snapshot to file and appends the corresponding WAL entry.
        // 存储 snap entry 到 WAL
        // snap entry 本质就是个 {index, term}
		if err = s.r.storage.SaveSnap(snap); err != nil {
			lg.Panic("failed to save snapshot", zap.Error(err))
		}
		if err = s.r.storage.Release(snap); err != nil {
			lg.Panic("failed to release wal", zap.Error(err))
		}

		// keep some in memory log entries for slow followers.
        // 为了较慢的 follower 考虑，不会将所有 applied 日志都删除，而是会保留一部分，这样给 follower 同步 entry
        // 的时候，就可以同步增量的 entry，否则只能通过发送 snapshot 的方式全量同步
		compacti := uint64(1)
		if snapi > s.Cfg.SnapshotCatchUpEntries {
			compacti = snapi - s.Cfg.SnapshotCatchUpEntries
		}

		err = s.r.raftStorage.Compact(compacti)
		if err != nil {
			// the compaction was done asynchronously with the progress of raft.
			// raft log might already been compact.
			if err == raft.ErrCompacted {
				return
			}
			lg.Panic("failed to compact", zap.Error(err))
		}
	})
}
```

### 如何从 WAL 恢复日志

当节点 crash 后，就需要根据 WAL 重建 raft log。具体流程如何呢？大体思路如下：
- 找最近的 snapshot entry，并获取 snapshot entry.index 之后的所有 entry 添加到 ratlog.storage 中
- 重置 raftlog.committed = firstindex - 1; raftlog.applied = firstindex - 1;

之后就是从 leader 接受 entry 以及 commited_index 来更新 follower 的 committed_index. <br>

WAL 还有一类类型为 HardState 的 entry，内部包含 commited_index、term 消息，当节点从 WAL 恢复的时候，还会基于最近的 HardState entry 来更新 commited_index。


### 已经 commit 但是 apply 失败，然后服务重启，重放 commit 有没有可能 apply 成功，这不就不一致了 ？

etcd 在 apply 的时候如果发生错误，对于**写**操作(只有写操作，才会有 entry)如果发生的错误是稳定的，比如 key 不存在、lease 不存在，那么即使重放该 entry，结果也一定是可预见的。而对于一些结果不稳定的错误，本次失败但下次可能成功的 entry，服务直接 panic，或者一些严重的服务内容的错误直接 panic。<br>

在 etcd 内部，我们可以见到很多发生错误直接 panic 的情况，一些场景下，要保证几个操作之间的原子性，如果中间某一步发生错误，也直接 panic。

## 日志复制

### progress

leader 为每一个 follower 维护了进度信息，包括 match: match 日志的 index, next_index: 从哪个 index 开始发送消息。<br>

因为 follower 可能异常退出、可能落后很多的 entry，因此 progress 维护了一个 state 字段表示 follower 的状态。分为三类：

- probe

    当 leader 与 follower 之间通信发生错误或者 leader 不知道 follower 的进度的时候，状态切换为 probe。任何时刻只能存在一个 MsgApp 消息，当 leader 收到 follower 的响应后，根据 follower 的复制进度状态迁移为 snapshot 或者 replication。

- snapshot

    如果 follower 落后 leader 很多（对应的日志已被 leader compact，在 raftlog 中不存在），此时，就只能发送 snapshot 给 follower，对应的复制状态就是 snapshot。


- replication

    follower 落后的日志都可以在 leader raftlog 中找到，表示落后不是很多也就是正常的复制状态，即 replication。

follower 的复制状态就在上面三个状态之间转移。

### 日志冲突

首先一个总原则：日志的每个 entry 包括两个要素：index、term。日志的 index 是递增的，日志的 term 也是递增的。<br>

leader 给 follower 发送 MsgApp 消息时，除了发送 entries，还会**发送第一个 entry 的前一个 entry 的 index 和 term**，follower 接收到 MsgApp 后，首先会判断这个特定 entry 的 index 和 term 能否匹配，如果不能即表示存在冲突。随后，follower 会在 MsgAppResp 中带上该 index 处的 term，leader 收到后，通过下面的逻辑调整下一次 MsgApp 的起始位置：
```go
// 从 index 回溯，直到某个 index 的 term <= term，因为 > term 的 index 一定是冲突的。
func (l *raftLog) findConflictByTerm(index uint64, term uint64) uint64 {
	for {
		logTerm, err := l.term(index)
		if logTerm <= term || err != nil {
			break
		}
		index--
	}
	return index
}
```

## server 从 raft 拿到一个 entry，怎么区分底层到底对应哪个 req 呢？

server 收到一个 client 的请求后，如果是一个写操作，会将该请求 Propose 给 raft 层，raft 层会将 req 转化为一个 entry，当该 entry 在集群中得到共识后，就会 apply 到 server，
可是这个时候，server 看到的是一个 []byte，怎么区分该 entry 到底对应哪种类型的 req 呢？<br>

如果直接把 req marshal 存到 entry 后，会丢失 req 类型信息，也就无法区分 req 类型。etcd 的做法是，会在 raw req 基础上，定义一个 InternalRaftRequest 类型，该类型定义如下：
```go
type InternalRaftRequest struct {
    Header                   *RequestHeader                            `protobuf:"bytes,100,opt,name=header,proto3" json:"header,omitempty"`
    ID                       uint64                                    `protobuf:"varint,1,opt,name=ID,proto3" json:"ID,omitempty"`
    Range                    *RangeRequest                             `protobuf:"bytes,3,opt,name=range,proto3" json:"range,omitempty"`
    Put                      *PutRequest                               `protobuf:"bytes,4,opt,name=put,proto3" json:"put,omitempty"`
    DeleteRange              *DeleteRangeRequest                       `protobuf:"bytes,5,opt,name=delete_range,json=deleteRange,proto3" json:"delete_range,omitempty"`
    Txn                      *TxnRequest                               `protobuf:"bytes,6,opt,name=txn,proto3" json:"txn,omitempty"`
    Compaction               *CompactionRequest                        `protobuf:"bytes,7,opt,name=compaction,proto3" json:"compaction,omitempty"`
    LeaseGrant               *LeaseGrantRequest                        `protobuf:"bytes,8,opt,name=lease_grant,json=leaseGrant,proto3" json:"lease_grant,omitempty"`
	// ...
}
```

可以看到内部每一个成员对应一种类型的请求。<br>

接着，在 server 处理 req 的入口处，会基于每一个 raw req，封装一个 InternalRaftRequest 请求，比如 DeleteRange 处理如下：
```go
func (s *EtcdServer) DeleteRange(ctx context.Context, r *pb.DeleteRangeRequest) (*pb.DeleteRangeResponse, error) {
	// 我们看到这里会包装生成一个 InternalRaftRequest，有什么作用呢？
	// sever 最后 apply 的数据是从 raft 层拿到的，如果这里不包装成 InternalRaftRequest，
	// server 就无法区分从 raft 获取到的 entry 到底对应哪个请求。
	// 而如果这么包装后，server unmarshal 后，哪个成员非空，那就对应哪个类型的请求。
	resp, err := s.raftRequest(ctx, pb.InternalRaftRequest{DeleteRange: r})
	if err != nil {
		return nil, err
	}
	return resp.(*pb.DeleteRangeResponse), nil
}
```

这样对 InternalRaftRequest marshal 后，其实保留了 raw req 的类型信息，后续 entry apply 到 server 的时候，只需要 unmarshal，然后依次编译每一个成员，取第一个非空的即可。

# leader 选举

## 线性读的时候，如果发生了 leader 选举，怎么处理

如果线性读的时候，发生了 leader 选举，当请求发送到新的 leader 的时候，leader 能处理线性读的前提是：**commited_index > 旧 leader 的 commited_index，这样才能保证线性读的语义**<br>

那么 etcd 是如何保证的呢？那就是 **新 leader 在当前 term 有任意一个 commit!**

```go
	case pb.MsgReadIndex:
		// only one voting member (the leader) in the cluster
		if r.prs.IsSingleton() {
			if resp := r.responseToReadIndexReq(m, r.raftLog.committed); resp.To != None {
				r.send(resp)
			}
			return nil
		}

		// Postpone read only request when this leader has not committed
		// any log entry at its term.
		if !r.committedEntryInCurrentTerm() {
			r.pendingReadIndexMessages = append(r.pendingReadIndexMessages, m)
			return nil
		}

		sendMsgReadIndexResponse(r, m)

		return nil
```

## term

term 类似于 kafka 中的 epoch(纪元)，每当发生 leader 切换的时候，term + 1。

## leader 切换流程

每个 follower 维护一个 election 超时时间，每当接收到来自 leader 的消息的时候，该超时时间重启及时，当超过 election 超时时间后，follower 会发起选举流程，自身状态切换为 candidate。<br>

自身的 term + 1，给自己投票，然后给集群中的所有其他节点(集群节点通过 progress 维护)发送投票消息，其他节点收到后，会根据以下条件判断是否投票：

- 自己有没有检测到 leader 失联
- 自己在当前 term 有没有投过票
- 发起选举的节点的 entry 是不是跟自己相同或者比自己新

只有条件都成立时，才给该节点投票，否则投反对票。当发起投票的节点接收到的赞成票 >= 集群 qurnum( n/2 + 1) 的时候，就切换为 leader，其他节点收到该 leader 的任意消息的时候，更改自身维护的 leader_id。如果大多数节点都投了反对票，则节点切换为 follower，并在 election timeout 后发起新一轮选举，除非在此期间收到了新 leader 的消息。<br>

如果同一时间有多个 follower 发起了 选举怎么办？这样选举成功的概率就比较低，紧接这会发起新的一轮选举，周而复始。<br>

etcd 的解决办法是，给 follower 的 election timeout 加一个随机时间，这样就可以大大提高选举的成功率。用最简单的办法解决最复杂的问题，应是我们的追求。

## preVote

preVote 就是预投票的意思，提前进行一次投票，如果投票可以赢得选举，才发起真正的投票。目的就是减少无效的投票行为。<br>

**我们要特别需要学习这种 预 的思想，在分布式事务的提交中也有个预提交的思路，为了尽量保证后面的操作成功，我们可以先预练一次**。

# 集群管理

- 一个集群是怎么搭建的, 如何添加/删除节点

集群中节点信息当然要在各个节点达成一致，etcd 也是通过日志的方式来同步集群变更信息的。每当集群中增加/删除 节点的时候，就会生成一个 ConfChange 类型的 entry，然后 apply 到集群的各个节点。<br>

其实在新建集群的时候，也会针对集群中的每个节点生成一个 entry，然后在集群所有节点 apply。这里就有一个问题，在搭建集群的过程中，是还没有 leader 的，而日志同步时通过 leader 发送给 follower 的方式进行的，那怎么同步呢？当节点 crash/net partition 的时候，leader 又是如何感知的呢？<br>

当集群中的节点启动的时候，是能知道 cluster 都有哪些节点的，只需要对每个节点生成一个 entry 然后 apply 即可，这样所有节点的 entry 信息都是一致，并不需要通过 leader 的方式同步。<br>

- 当集群中新增节点的时候并且节点成功启动后，或者一个异常的节点重启后，leader 节点是如何感知到这个节点的呢？

leader 通过 progress 来表示 follower 的同步状态。节点之间是通过 rafthttp 进行通信的，当 leader 通过 rafthttp 给 follower 发送消息的时候，如果此时出错，rafthttp 调用 raft 接口标记该 follower 不可达，也就是设置 progress.state 为 probe(探测)。此后，leader 只给该 follower 发送 heartbeat 消息。<br>

当节点重启后，某个时刻接收到 leader 的 heartbeat 消息，然后发送 heartbeat resp 消息给 leader，leader 收到后根据该节点的复制进度更改 progress.state 为 replicate 或者 snapshot。后续进行正常的同步流程。

综上，follower 的存活是通过 leader 的 heartbeat 方式判断的。

- 每个成员的 id 是如何分配的 ？

根据节点的 URLS 和 集群 token 信息来生成一个 id，通过此种方式，在任意一个节点上，生成的某个节点的 id 都是一样的。<br>

我们需要特别注意这种思路，**一些信息，并不需要通过在节点之间传递的方式共享，完全可以在每个节点上使用相同的 input + 相同的算法 生成相同的 output**
