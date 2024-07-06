描述 etcd 中 mvcc 相关内容。<br>

etcd 并没有直接使用 bbolt 的事务，而是基于 bbolt 实现的。

# 若干问题

- compactor 的工作原理是什么 ？
- consistent index 是什么时候持久化的 ？

## 事务


### 整体架构

我们先来看看 etcd server 的整体架构。

-------------------------------------
|               entries             |
-------------------------------------
                  |
                  | apply
                  |
                  \/
--------------------------------------
|      创建读写事务 TxnWrite/Read      |
---------------------------------------
                  |
                  | 读/写 store
                  |
                  \/
---------------------------------------
|               store                 |
---------------------------------------
|       B Tree Index | bbolt db       |
---------------------------------------
                            | 
                            | 
                            |
                            \/
                    -----------------------------
                    | 一个全局 bbolt 写事务：batch_tx|
                    -----------------------------
                    | 一个全局 bbolt 读事务：read_tx |
                    -----------------------------

实际上， etcd 中的事务有三类：客户端 stm 实现的事务、server 端的 TxnWrite/TxnRead、bbolt 中的读写事务。TxnWrite/TxnRead 基于 bbolt 事务实现，stm 基于 TxnWrite/TxnRead 实现。<br>

关于 bbolt 的事务，我们已经在 bboltdb.md 中说明，本文我们重点描述其他两类事务。

### TxnWrite/TxnRead

当 server apply 一个 entry 的时候，会根据读写操作创建对应的 TxnWrite/TxnRead。实际上，在任意时刻 store 都会引用一个 bbolt 写事务也就是所谓的 batchTx，一个 bbolt 的读事务。TxnWrite/TxnRead 都是基于这两个全局的 bbolt 事务实现的。

- batchTx

    实际上，我们完全可以针对每个写请求，创建一个 bbolt 写事务。但是这样，性能会比较差，涉及到大量的磁盘 i/o，需要从磁盘获取 meta page 创建事务，在提交的时候，也需要持久化。<br>

    为了提高性能，store 底层一个时间段只创建一个 bbolt 写事务，所有的 TxnWrite 的写操作都通过 bbolt 写事务完成。当写入 TxnWrite 达到一定值或者定时或者存在删除操作时，才提交 bbolt 写事务，并创建一个新的 bbolt 写事务，用于后续的 TxnWrite。<br>

    这里带来的一个问题是：因为真正的持久化是异步的，当客户看到写操作成功时，对应的数据可能并没有真正的持久化。如果此时 server crash 了，那怎么办呢？<br>

    实际上没有问题，因为 consistent index 的存在，当 server restore 的时候，就会基于 consistent index replay 已提交但未持久化的 entry。<br>

    - 实现
    
        实现路径为：mvcc/backend/batch_tx.go。
```go
        type batchTxBuffered struct {
            batchTx
            buf                     txWriteBuffer // 当 TxnWrite commit的时候，将结果从此 buf 写入到 readTx 对应的 buf 中。
            pendingDeleteOperations int // 只要有删除操作，就需要立即 commit，因为删除操作结果不会存到 buffer 中
        }

        type batchTx struct {
            sync.Mutex // 互斥锁用来保护 tx，tx 的写入、commit 都是互斥操作
            tx      *bolt.Tx
            backend *backend

            pending int // 对应的 TxnWrite 数量
        }

 ```

- readTx

    同样，TxnRead 也是基于 bbolt 的读事务来实现的。需要注意的是，bbolt 的读/写事务都是在 bbolt 写事务 commit 的时候，重新创建的，之所以同时创建，是因为新的 bbolt 读事务可以读到 bbolt 写事务的变更。<br>

    这里有个问题，因为 bbolt 读/写事务同时创建的，所以读事务是无法获取到写事务的变更的(bbolt 的 Copy On Write)。为了保证线性读，在 TxnWrite 事务提交的时候，TxnRead 需要能读到 TxnWrite 的变更。那怎么办呢？<br>

    readTx 的实现中包含一个 buffer，该 buffer 就是用来存储 TxnWrite 的结果的。TxnWrite 在 End 的时候，除了将变更写入到 bbolt 写事务，还会将结果写入到此 buffer 中。TxnRead 在读取数据的时候，会首先从该 buffer 读，未命中后才会基于 bbolt 读事务读取数据。当 bbolt 写事务 commit  的时候，会 reset 该 buffer。<br>

    而这就是 buffer 存在的意义。

    - 实现
  
        readTx 的实现对应的路径为：mvcc/backend/read_tx.go，定义如下：
 ```go
            type baseReadTx struct {
                // mu protects accesses to the txReadBuffer
                mu  sync.RWMutex
                buf txReadBuffer // buffer，存储 TxnWrite 写的结果

                tx      *bolt.Tx // 对应的 bbolt 读事务
                // ...
            }
```

### 如何提高 txWrite 和 txRead 的并发度

我们知道 txWrite 和 txRead 的执行过程中，都会涉及一些共享资源，比如 index tree, read buffer 等。如何避免读写事务的冲突呢？<br>

一种简单的思路是，维护一个 store 粒度的互斥锁，读写事务都需要占有这个锁。显然，这种方式的并发度太低了，那应该怎么办呢？<br>

虽然共享资源是免不了加锁的，但我们可以将锁的粒度变小。比如 index tree 定义一个锁，read buffer 定义一个锁。这样在访问具体的资源的时候，才需要互斥。提高系统并发度。

### store

store 包括：index 和 backend 两个部分。etcd 的 mvcc 就是基于 index 实现的。

#### index

etcd 的 store 会维护一个全局的逻辑时钟，也就是版本号：revision。当 etcd 有写入操作的时候：新增、变更、删除，都会生成一个新的版本，并以此版本号为 key，以业务的 key、value、create_revision、mod_revision 等信息为 value 存入到 bbolt 中。<br>

而这就是 etcd 实现 mvcc 特性的关键，而 mvcc 是实现快照读的关键。那么 store 是如何存储 key 到 revision 的映射呢？就是通过 B Tree，这是一个存内存数据结构，当 server restart 的时候，基于 bbolt 重建 B Tree。

#### revision

revision 本质就是个逻辑时钟。在 store 中，维护了一个 `currentRev` 就是最近完成的 TxnWrite 对应的主版本号。每个 TxnWrite 会在 currentRev 基础上生成一个新的主版本号：newRev = currentRev + 1，并在 TxnWrite 提交的时候，更新 currentRev。TxnWrite 中的每个写操作会对应一个从 0 开始的子版本号。因此完整的 revision 格式为：{mainRev, subRev}

### STM

还有一类就是基于 etcd server 在客户端实现的事务，etcd 在 `client/v3/concurrency/stm.go` 中已经给我们做了封装。

- client txn api
  
    etcd 给我提供的 txn api 格式如下：
```go
// Txn is the interface that wraps mini-transactions.
// 这是一个完整的使用例子。站在 server 角度看，这就是一个 entry 而已
//	Txn(context.TODO()).If(
//	 Compare(Value(k1), ">", v1),
//	 Compare(Version(k1), "=", 2)
//	).Then(
//	 OpPut(k2,v2), OpPut(k3,v3)
//	).Else(
//	 OpPut(k4,v4), OpPut(k5,v5)
//	).Commit()
type Txn interface {
	// If takes a list of comparison. If all comparisons passed in succeed,
	// the operations passed into Then() will be executed. Or the operations
	// passed into Else() will be executed.
	If(cs ...Cmp) Txn

	// Then takes a list of operations. The Ops list will be executed, if the
	// comparisons passed in If() succeed.
	Then(ops ...Op) Txn

	// Else takes a list of operations. The Ops list will be executed, if the
	// comparisons passed in If() fail.
	Else(ops ...Op) Txn

	// Commit tries to commit the transaction.
    // TxnResponse 包括：if 条件是否成立、以及 Then/Else 每个 Op 的结果
	Commit() (*TxnResponse, error)
}

```

- 为什么还要提供 stm

    其实就是为了方便。将一些常见的操作基于 client txn api 封装在 stm 中，对外提供更加简单的 api。封装的操作包括：
    - 根据 get/write 操作构建 Cmp
    - 事务 Commit 失败(if 条件失败)下的重试

- 隔离级别
  
stm 实现了四种隔离级别：
```go
const (
	// SerializableSnapshot provides serializable isolation and also checks
	// for write conflicts.
	SerializableSnapshot Isolation = iota
	// Serializable reads within the same transaction attempt return data
	// from the at the revision of the first read.
	Serializable
	// RepeatableReads reads within the same transaction attempt always
	// return the same data.
	RepeatableReads
	// ReadCommitted reads keys from any committed revision.
	ReadCommitted
)
```

我们分别看看每种隔离级别是怎么实现的：

- SerializableSnapshot(默认)

    - 读取

        在首次读取数据结束后，会记录对应的 revision，后续所有的读取操作都基于此 revision。

    - 写入
    
        事务中的写入操作，并不会立即发生，而是会存放到一个称为 `wset` 的结构中。一来是为了让 Get 操作获取本事务写入的数据，二是最后可以基于 wset 生成 Then Ops。

    - IF 条件
    
        IF 条件包括两部分：
        - 读取的数据的 mod_revision 没有发生变化
        - 写入的数据的 mod_revision < init_revision

        也就是除了判断读取的数据期间没有发生变化，写入的数据也没有发生变化。

    - Then
    
        将 wset 操作转化为 Ops。

    - Else

        为了后续的重试，在 IF 条件失败的时候，会再次读取目标 key。stm 会将读取操作维护到 `rset` 中。Else 对应的 Ops 由 rset 转化而来。

- Serializeable

    与 SerializeableSnapshot 类似，只不过 if 条件只检测读数据是否发生变更。

- RepeatableReads

    与前两者不同，已经读取到的值会缓存下来，未读取到的 key，会读取最新版本的数据并缓存下来。同样 if 条件只检测读取到的数据是否有变化。

## compactor

- 为什么要压缩

    压缩指的是删除 <= 某个特定 revision 的数据，当然如果某个 key 的 latestrevision <= target_revision，这个 key 只会保留 latest_revision 的内容。如果 latest_revison 对应的是删除操作，就会删除该 key. <br>

    所以压缩的目的就是删除旧版本的数据，防止 bbolt 过大。

- 压缩时机

    - 用户主动压缩
    - server 配置定时压缩
    - server 配置版本压缩 

- 如何压缩

    压缩过程需要遍历整个 index tree，删除多余的数据，同时还需要删除 bbolt 对应的数据。当涉及的数据很多的时候，必然会影响到正常的读写事务，如何尽量降低对普通的读写事务的影响呢？<br>

    对于 index tree，核心思路是将 tree 维度的锁降低到 tree item 维度的锁。实现方法是先 clone 一个 index tree，然后遍历该 clone tree，对于每一个 item，加 origin index tree 的锁，调整完毕后，释放锁。在压缩 index 的期间，origin tree index 可能会插入一些新的 item，**但这些 item 对应的 revision 一定比 compact_revision 新，所以我们没必要处理该数据。**<br>
  
  - 如何 clone index tree 呢？
  
    通常的思路就是加一个 tree 维度的锁，然后逐个 item copy，copy 完之后，释放锁。但是，这种方式 copy 性能比较差，而且 copy 期间，会阻塞其他正常的读写请求。<br>
    
    而 etcd 引用的 btree 开源库，用了一个很妙的解法：**tree 是只读的，如果要修改，那就 copy-on-write，这样 clone 的时候只要获取到 root 信息即可**。<br>
  
    btree clone 的代码如下，我们需要重点关注 comment 部分：
```go
// Clone clones the btree, lazily.  Clone should not be called concurrently,
// but the original tree (t) and the new tree (t2) can be used concurrently
// once the Clone call completes.
//
// The internal tree structure of b is marked read-only and shared between t and
// t2.  Writes to both t and t2 use copy-on-write logic, creating new nodes
// whenever one of b's original nodes would have been modified.  Read operations
// should have no performance degredation.  Write operations for both t and t2
// will initially experience minor slow-downs caused by additional allocs and
// copies due to the aforementioned copy-on-write logic, but should converge to
// the original performance characteristics of the original tree.

func (t *BTree) Clone() (t2 *BTree) {
	// Create two entirely new copy-on-write contexts.
	// This operation effectively creates three trees:
	//   the original, shared nodes (old b.cow)
	//   the new b.cow nodes
	//   the new out.cow nodes
	cow1, cow2 := *t.cow, *t.cow
	out := *t
	t.cow = &cow1
	out.cow = &cow2
	return &out
}
```

当压缩完 index tree 后， 我们获得一个 valid revision 集合。然后会加 batchTx 的锁，更改 bbolt。但这里为了降低对正常读写事务的影响，通过分批的方式来压缩 bbolt，每次只压缩指定数量的数据。 

## consistent index

etcd 会在 bbolt 中持久化存储：consistent index 和 consistent term。持久化的时机是当 batchTx commit 的时候。
