描述 bboltdb 的相关细节。

# 若干问题

- 磁盘文件格式 ？
- 读写事务的并发性、隔离性？

# 磁盘

我们知道 bboltdb 是通过 `B+` 树在磁盘上存储文件的。

## 页格式

- header

    不同的页格式是不尽相同的，但是所有的页都有一个 header，其核心字段为：

    ```go
        struct header {
            id int // 页 id
            flag int // 页类型
            count int // 页包含 item 个数
            overflow int // 溢出页数量
            dataPtr int // 存放数据的起始地址
        }
    ```

- 溢出页

    为了不让 B+ 树太高，我们通常会限制 B+ 树节点中 key 的最小数量。但即便页中存储最小数量的 key，如果 key 对应的 value 很大，就需要额外的页来存储数据。<br>

    在 bbolt 中，溢出页是一系列连续的页。在 页 header 中，通过 overflow 字段表示溢出页的数量。

- fillPercent

    这个指标是什么含义呢？<br>

    表示页的填充百分比，当页填充 > fillPercent 的时候，就需要页分裂。该指标取值范围为 [0.1, 0,9]，默认为 0.5.

## 页分类

分为四类：meta、freeList、branch、leaf


- meta

    在 bbolt 中存在这**两个**meta 页。为什么是两个，我们在写事务一节中描述。meta 有什么作用呢 ？<br>

    meta 代表了 bbolt 的**一致性状态**。每当有写事务 commit 的时候，就会更新 meta 页为一个新的状态，从而完成 db 一致性状态的迁移。<br>

    那么 meta 页都包含什么内容呢？核心字段如下：
    ```go
    type meta struct {
        magic    uint32
        version  uint32 // bbolt 版本，该字段用于版本管理，方便 db 版本的升级
        pageSize uint32 // bbolt 页大小
        flags    uint32 
        root     bucket // 根 bucket 对应的 pageID
        freelist pgid // 管理 free 页的 pageID
        pgid     pgid // db 文件最大页对应的 pgID 
        txid     txid // 最近成功提交的写事务 id，也可以理解为一个逻辑时钟
        checksum uint64 // 用于校验 meta 页完整性的，因为在事务提交更新 meta 的时候，有可能 meta 写了一半进程就 crash 了
    }
    ```

- free
  
    该页用来管理 bbolt 中的空闲页，存放空闲页的页号，如果空闲页很多，就通过溢出页来存放空闲页号。

- branch & leaf

    其实就是 B+ 树中的非叶子节点和叶子节点。为了实现二分查找，页内部的每个 item 都会对应一个固定大小的 element，element 就是根据 key 排序的，之所以固定大小，就是为了方便二分查找。

    - branchElement

       我们都知道，B+ 树非叶子节点是不存储 value 的。<br>

       ```go
        type branchPageElement struct {
            pos   uint32 // 存放 key 的起始地址
            ksize uint32 // key 的大小
            pgid  pgid // 对应下层页的 id
        }
       ``` 

    - leafElement

        ```go
        type leafPageElement struct {
            flags uint32 // 因为 root bucket 和 普通 bucket 的叶子节点存储的内容不同，这个 flag 用于区分存储什么类型的数据
            pos   uint32 // key 与 value 存放在一起，pos 值起始地址
            ksize uint32 // key 大小
            vsize uint32 // value 大小
        }
        ```

## bucket

bucket 在 bbolt 中对应一个 B+ 树，都我们需要存放不同内容的时候，就可以创建一个单独的 bucket。bucket 对象定义本质就是 B+ 树 root page 的 id.<br>

```go

    type bucket struct {
        root     pgid   // page id of the bucket's root-level page
        sequence uint64 // 暂时忽略该字段
    }
```

buckets 的管理就是通过 root bucket 完成的，该 bucket 就是用来存储 bucket 的 bucket 的，该 bucket 叶子节点中的 key 对应 bucket 的 name，value 对应 bucket struct{}.


## 只增不减

bbolt 的一个特性就是**文件只增不减**，空闲的页会交给 free page 管理，不会释放。这就是为什么我们删除了数据后，为什么 db 文件并没有变小的原因。

# 事务

bbolt 也是支持事务的，事务隔离级别个人觉得是 **串行化**。

## 读事务

bbolt 允许同时存在多个读事务。每个读事务被创建的时候，会拷贝一份 meta 页内容，这个 meta 代表了 db 当前的快照。后续事务所有的读操作，都是基于该 meta 进行的。

## 写事务

**bbolt 任意时刻只允许存在一个写事务。**这是通过 DB 内存对象中的互斥锁实现的。<br>


- copy on write

    bbolt 实现读/写事务隔离的方法就是通过 `Copy On Write`，当写事务更改数据的时候，就会基于原来的 page 创建一个 node 数据结构，然后所有的变更写入到 node 中，在提交的时候，对 node 进行重平衡，然后转化为对应的 page，写入到数据库。

- 事务提交

    需要特别注意的是，写事务的提交涉及到多个动作，包括：写入脏页、写入 free 管理页、写入 meta 页。我们必须要保证这几步操作的原子性，只有全部写入成功后，事务提交才算成功。<br>
    
    那么，如何保证原子性呢？如果写入失败了怎么办呢？<br>

    bbolt 的解决办法是:<br>

    调整这几步的写入顺序，将 meta 的写入放到最后，meta 的成功写入，才代表 bbolt 完成状态的一致性迁移，如果 meta 写入失败，分两种情况讨论。如果新的 meta 还没有写入，其他事务还是能基于旧的 meta 执行操作，当前事务的所有变更都不会影响到其他事务。如果新的 meta 写入了一半就失败了呢？这样其他事务还是会读取到不完整的 meta 页，怎么解决呢？

   **这就是 bbolt 设计两个 meta 的原因，当有一个 meta 不完整的时候，可以使用另外一个完整的 meta，当然 meta写的写入是更新原来 txid 较小的 meta 页，这样即使写入失败，其他事务仍然能够基于最近成功提交的事务继续执行操作** 

- 事务回滚

    事务回滚实际也没啥特别的操作。因为写事务执行中会涉及到 freeList 内存数据结构的变化，回滚页仅仅是更新 freeList 确保 freeList 处于一个一致性状态。

- 页释放  

    当一个写事务提交的时候，事务修改的页对应的原始页并不会立即释放，为什么以及什么时候会释放呢？

    - 为什么
    
        因为 Copy On Write 特性，当写事务成功提交后，脏页对应的原始页可能还有读事务在使用，因此不能立即释放。

    - 真正释放时机

        只有比写事务小的所有读事务都结束的时候，原始页才能真正释放。bbolt 中，会维护一个读事务集合，同时在 freeList 内存数据结构中维护了每个写事务对应的原始页集合。<br>

        在创建新事务的时候，首先获取当前 db 中进行中的最小事务 id - 1，然后遍历 freeList 中所有未释放原始页的写事务，将写事务 id <= (当前最小读事务 id - 1) 的事务对应的原始页释放。<br>

        不过个人觉得，最准确的时机应该是读事务结束的时候判断。放在写事务中，是为了提高读性能吗？<br>

        所以，**我们要尽量避免长的读/写事务，读事务太大，就会导致 bbolt 快速增长。**

- 页分配

    当当前的 freePage 不够时候，bbolt 就会 unmap 掉原来的映射，并通过 mmap(memory map) 的方式扩展 db 文件大小. 这里的问题是原来的 mmap 区可能还有读事务在访问，那什么时候才能 unmap 呢？

    bbolt 会通过一把 db 的全局读写锁来保证 mmap 的互斥，每当创建一个读事务的时候，就会持有一个读锁，当需要重新 mmap() 的时候，会获取写锁，进而保证两者的同步。

# B+ Tree rebalance

TODO: 看看写事务提交的时候是如何 rebalance 的