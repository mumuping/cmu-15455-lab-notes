# lab 4

## 1. Task #1 - Lock Manager

1. Lock Manager 的基本思想：在正在运行的事务内部会有一个数据结构，用于保存这个事务当前已经获取的锁，然后 LockManager 就是来管理这个事务中的这个数据结构，即授予该事务锁、阻塞该事务（当事务获取锁时）、或 abort 该事务（当事务获取锁时）。举个例子，比如一个事务想要获取某个 table 的某个 tuple，那么它先会向 LockManager 申请锁，只有得到锁后才能够获取到该 tuple。 

   > The basic idea of a LM is that it maintains an internal data structure about the locks currently held by active transactions. Transactions then issue lock requests to the LM before they are allowed to access a data item. The LM will either grant the lock to the calling transaction, block that transaction, or abort it.
2. Lock Manager 的任务：根据事务的隔离等级来授权和释放事务的锁。

   > This task requires you to implement a table-level and tuple-level LM that supports the three common isolation levels: READ_UNCOMMITED, READ_COMMITTED, and REPEATABLE_READ. The Lock Manager should grant or release locks according to a transaction's isolation level.

   需要实现以下四个函数：

   ```cpp
   // 表
   LockTable(Transaction, LockMode, TableOID)
   UnlockTable(Transction, TableOID)
   // 行
   LockRow(Transaction, LockMode, TableOID, RID)
   UnlockRow(Transaction, TableOID, RID)
   ```

3. 已提供的信息：`include/concurrency/transaction.h`中提供了事务的相关信息，比如事务的隔离等级、事务当前状态、事务已拥有的不同锁的集合等。`include/concurrency/transaction_manager.h`中提供了事务 create、commit、abort、BlockAllTransactions、ResumeTransactions 等。

4. Task1 的一些提示：

    > While your Lock Manager needs to use deadlock detection, we recommend testing and verifying the correctness of your lock manager implementation first without any deadlock handling before adding detection mechanisms.
    >
    > 先不用考虑死锁检测
    >
    > You will need some way to keep track of which transactions are waiting on a lock. Take a look at LockRequestQueue class in lock_manager.h
    >
    > 需要记录一下哪个事务在等待什么锁
    >
    > When do you need to upgrade a lock? What operations on the LockRequestQueue is needed when you need to update a table/tuple lock?
    >
    > 在处理锁请求队列时，什么时候应该升级锁，怎么升级？
    >
    >    While upgrading, only the following transitions should be allowed:
    >
    >    IS -> [S, X, IX, SIX]
    >
    >    S -> [X, SIX]
    >
    >    IX -> [X, SIX]
    >
    >    SIX -> [X]
    >
    > You will need some way to notify transactions that are waiting when they may be up to grab the lock. We recommend using std::condition_variable provided as part of LockRequestQueue. 
    >
    > 使用条件变量阻塞还未得到授权的请求；当某个请求得到授权后，要通知后面还在等待被处理的请求。
    >
    > You should maintain the state of a transaction. For example, the states of transaction may be changed from GROWING phase to SHRINKING phase due to unlock operation (Hint: Look at the methods in transaction.h)
    >
    > 二阶段锁。
    >
    > You should also keep track of locks acquired by a transaction using *_lock_set_ so that when the TransactionManager wants to commit/abort a transaction, the LM can release them properly.
    >
    > 需要保存事务请求的锁，这样当事务被 commit 或者 abort 的时候，能够清理这些还在请求中的锁。
    >
    > Setting a transaction's state to ABORTED implicitly aborts it, but it is not explicitly aborted until TransactionManager::Abort is called. You should read through this function and provided tests to understand what it does, and how your lock manager is used in the abort process.
    >
    > 注意事务是怎么 abort 的。

5. 加锁需考虑的点：

    ```cpp
    /**
    * [LOCK_NOTE]
    *
    * GENERAL BEHAVIOUR:
    *    Both LockTable() and LockRow() are blocking methods; they should wait till the lock is granted and then return.
    *    If the transaction was aborted in the meantime, do not grant the lock and return false.
    *
    * MULTIPLE TRANSACTIONS:
    *    LockManager should maintain a queue for each resource; locks should be granted to transactions in a FIFO manner.
    *    If there are multiple compatible lock requests, all should be granted at the same time
    *    as long as FIFO is honoured.
    *
    * SUPPORTED LOCK MODES:
    *    Table locking should support all lock modes.
    *    Row locking should not support Intention locks. Attempting this should set the TransactionState as
    *    ABORTED and throw a TransactionAbortException (ATTEMPTED_INTENTION_LOCK_ON_ROW)
    *
    *
    * ISOLATION LEVEL:
    *    Depending on the ISOLATION LEVEL, a transaction should attempt to take locks:
    *    - Only if required, AND
    *    - Only if allowed
    *
    *    For instance S/IS/SIX locks are not required under READ_UNCOMMITTED, and any such attempt should set the
    *    TransactionState as ABORTED and throw a TransactionAbortException (LOCK_SHARED_ON_READ_UNCOMMITTED).
    *
    *    Similarly, X/IX locks on rows are not allowed if the the Transaction State is SHRINKING, and any such attempt
    *    should set the TransactionState as ABORTED and throw a TransactionAbortException (LOCK_ON_SHRINKING).
    *
    *    REPEATABLE_READ:
    *        The transaction is required to take all locks.
    *        All locks are allowed in the GROWING state
    *        No locks are allowed in the SHRINKING state
    *
    *    READ_COMMITTED:
    *        The transaction is required to take all locks.
    *        All locks are allowed in the GROWING state
    *        Only IS, S locks are allowed in the SHRINKING state
    *
    *    READ_UNCOMMITTED:
    *        The transaction is required to take only IX, X locks.
    *        X, IX locks are allowed in the GROWING state.
    *        S, IS, SIX locks are never allowed
    *
    *
    * MULTILEVEL LOCKING:
    *    While locking rows, Lock() should ensure that the transaction has an appropriate lock on the table which the row
    *    belongs to. For instance, if an exclusive lock is attempted on a row, the transaction must hold either
    *    X, IX, or SIX on the table. If such a lock does not exist on the table, Lock() should set the TransactionState
    *    as ABORTED and throw a TransactionAbortException (TABLE_LOCK_NOT_PRESENT)
    *   多层级锁，比如要对某行加锁时，要保证该事务已经获取了表上的相应锁，然后才能对行进行加锁。
    *
    *
    * LOCK UPGRADE:
    *    Calling Lock() on a resource that is already locked should have the following behaviour:
    *    - If requested lock mode is the same as that of the lock presently held,
    *      Lock() should return true since it already has the lock.
    *    - If requested lock mode is different, Lock() should upgrade the lock held by the transaction.
    *
    *    A lock request being upgraded should be prioritised over other waiting lock requests on the same resource.
    *
    *    While upgrading, only the following transitions should be allowed:
    *        IS -> [S, X, IX, SIX]
    *        S -> [X, SIX]
    *        IX -> [X, SIX]
    *        SIX -> [X]
    *    Any other upgrade is considered incompatible, and such an attempt should set the TransactionState as ABORTED
    *    and throw a TransactionAbortException (INCOMPATIBLE_UPGRADE)
    *
    *    Furthermore, only one transaction should be allowed to upgrade its lock on a given resource.
    *    Multiple concurrent lock upgrades on the same resource should set the TransactionState as
    *    ABORTED and throw a TransactionAbortException (UPGRADE_CONFLICT).
    *   锁升级的意思是：请求队列中已经存在该事务了（该事务之前已经请求了该资源），然后现在该事务又请求相同的资源
    *   （这次该事务请求的 lock_mode 可能与前一次请求的 lock_mode 不同），那么就需要判断是否需要进行锁升级。
    *   如果这次请求的 lock_mode 与上次请求的 lock_mode 一样，那么直接返回 true，因为该事务已经请求了该资源的 lock_mode。
    *   如果不一样，则需要按照上面的升级规则进行升级，若不能升级则抛出异常。
    *
    *   此外，上面还提到，一次只能有一个锁升级，当发现已经有锁在升级时，抛出异常（在请求队列中有提供 upgrading_ 字段，表示
    *   当前正在升级的事务 id。只有该事务被授权后，才能重置该字段）。
    *
    *   那么怎么进行锁升级呢？上面有提到如果一个事务进行锁升级，那么它的优先级是高于那些正在等待锁授权请求的事务。具体而言，
    *   首先从请求队列中找到该事务的前一次请求，并从请求队列中拿出来，然后创建一个新的请求，该请求是升级后的请求，然后再将
    *   该请求重新放入队列中，其位置应该是所有未被授权的请求的最前面，也就是说下一个待处理的请求就是该升级后的请求。然后将
    *   请求队列中的 upgrading_ 字段设置为该事务，表示正在升级该事务请求。然后由于该新请求是下一个待处理的请求，那么我们
    *   现在就需要尝试对它授权，若授权成功，则重置 upgrading_ 字段，表示升级完成，否则我们就需要等待，直到授权成功，或者事务
    *   自己 abort（注意 abort 的时候需要冲请求队列中删除这个新的请求）（如何授权？看下面第 6 点总结）。
    *
    *   上面我们还需要考虑一件事：该事务的前一次请求是否已经被授权了？如果未被授权，那么按上面方式处理即可；如果已经被授权了，
    *   那么我们就需要从该事务所保存的已经获取的锁集合中，删除上一次获取的锁，把锁回收，然后再按上面方式重新申请。
    *

    *
    * BOOK KEEPING:
    *    If a lock is granted to a transaction, lock manager should update its
    *    lock sets appropriately (check transaction.h)
    */
    ```

6. 综上所述，表加锁的工作流程大致应该是这样的：LockManager 会有以下数据结构用于保存请求哪个表/行的锁请求队列。当一个锁请求到达 LockManager 时，会检查它访问的表/行是否存在请求队列。如果不存在，说明当前没有事务请求该表/行，这时需要创建一个对应的请求队列，并将该请求加入到请求队列中，然后授予该请求锁。当请求队列存在时，说明当前有请求该表/行的事务队列，这时需要检查请求队列中该事务前面是否已经申请到了锁，考虑是否能够进行锁升级。若请求队列中不存在该事务，则将该请求加入到请求队列的队尾，然后判断如果这个请求之前所有的请求都被授予了，并且这个请求所需的锁与前面请求的锁是相容的，那么这个请求也会被授予，否则这个事务只能等待（这就是如何授权，也就是说在授权是按照 FIFO 进行的，当授权某个事务时，一定要保证它请求的锁类型要兼容前面**所有**事务请求的锁类型）。（在整个过程中，一定要记得处理事务本身内部所保存的一些锁集，比如授权时应该在事务内部锁集添加这个锁、锁升级时若原前一个请求已经被授予，则要从锁集中删除这个锁，等等）

    ```cpp
    /** Structure that holds lock requests for a given table oid */
    std::unordered_map<table_oid_t, std::shared_ptr<LockRequestQueue>> table_lock_map_;
    /** Structure that holds lock requests for a given RID */
    std::unordered_map<RID, std::shared_ptr<LockRequestQueue>> row_lock_map_;
    ```

7. 注意：让事务的请求等待，可以利用请求队列提供的条件变量解决，也就是说，当我们检查到不能给该事务请求授权时，我们可以调用`request_queue->cv_.wait(lk);`挂起该任务，等待被唤醒。如果被唤醒，首先应该检查的是该事务是否 abort，因为事务被 abort 和 commit 时也会唤醒该条件变量（存疑），然后再检查一遍是否可以 grant。此外，由于条件变量可能会被伪唤醒，因此要将该逻辑写在一个循环中。注意不能合并写成`request_queue->cv_.wait(lk, [&](){return IsGrantLock(request_queue, upgrade_request);})`，因为事务 abort 时 IsGrantLock 可能不会返回 true，导致一直等待。最后，在处理完请求队列的一个请求后，也应该调用该条件变量去唤醒被阻塞的、锁兼容的请求。当然如果是 X 类型的锁，就不用唤醒其他的请求了，因为没有跟它兼容的锁。

8. 行加锁的工作流程：与表加锁几乎一样，只是要在加行锁前先检查该事务是否已经获取了相应的表锁，如果没有获取相应的表锁，则抛出异常。此外，应注意最开始的检查与表加锁还略有差别，比如在行上不能加 IS、IX、SIX 锁，等等情况。

9. 解锁需要考虑的点：

    ```cpp
    /**
    * [UNLOCK_NOTE]
    *
    * GENERAL BEHAVIOUR:
    *    Both UnlockTable() and UnlockRow() should release the lock on the resource and return.
    *  解锁，也就是说要从 LockManager 所持有的关于这个 resource 的请求队列中删除事务已经被授权的请求
    *
    *    Both should ensure that the transaction currently holds a lock on the resource it is attempting to unlock.
    *  要保证解锁之前，该事务确实有持有该锁
    *
    *    If not, LockManager should set the TransactionState as ABORTED and throw
    *    a TransactionAbortException (ATTEMPTED_UNLOCK_BUT_NO_LOCK_HELD)
    *
    *    Additionally, unlocking a table should only be allowed if the transaction does not hold locks on any
    *    row on that table. If the transaction holds locks on rows of the table, Unlock should set the Transaction State
    *    as ABORTED and throw a TransactionAbortException (TABLE_UNLOCKED_BEFORE_UNLOCKING_ROWS).
    *  在解锁表时，要检查该事务是否还持有该表行的锁
    *
    *    Finally, unlocking a resource should also grant any new lock requests for the resource (if possible).
    *  解锁后，要通过请求队列中的条件变量唤醒等待被授权的请求
    *
    * TRANSACTION STATE UPDATE
    *    Unlock should update the transaction state appropriately (depending upon the ISOLATION LEVEL)
    *    Only unlocking S or X locks changes transaction state.
    *  根据隔离级别更新事务状态
    *
    *    REPEATABLE_READ:
    *        Unlocking S/X locks should set the transaction state to SHRINKING
    *
    *    READ_COMMITTED:
    *        Unlocking X locks should set the transaction state to SHRINKING.
    *        Unlocking S locks does not affect transaction state.
    *
    *   READ_UNCOMMITTED:
    *        Unlocking X locks should set the transaction state to SHRINKING.
    *        S locks are not permitted under READ_UNCOMMITTED.
    *            The behaviour upon unlocking an S lock under this isolation level is undefined.
    *
    *
    * BOOK KEEPING:
    *    After a resource is unlocked, lock manager should update the transaction's lock sets
    *    appropriately (check transaction.h)
    */
    ```

10. 解锁，也就是说要从 LockManager 所持有的关于这个 resource 的请求队列中删除事务已经被授权的请求。当然在解锁之前也应该进行相应的检查，此外解锁还应该根据隔离级别来更新事务状态。最后解锁后，事务内部的锁集也应该更新。

## 2. Task #2 - Deadlock Detection

1. 任务：实现一个死锁检测算法，然后创建一个后台线程，周期性地运行该算法。首先构造一个有向图（如果事务 A 等待事务 B，则 A 到 B 有一条有向边）；然后利用 dfs 算法检测该有向图是否有环，如果存在环，则返回环中最年轻的事务（txn id 最大的）。
2. 需要注意的点：
   1. 后台线程每次被唤醒运行时，都需要重新构建这个有向图，因此这个有向图是不用被保存的；
   2. 该 dfs 算法是确定的，也就是说每次 dfs 选择的下一个节点是子节点中 txn id 最小的那一个；
        > Your DFS Cycle detection algorithm must be deterministic. In order to do achieve this, you must always choose to explore the lowest transaction id first. This means when choosing which unexplored node to run DFS from, always choose the node with the lowest transaction id. This also means when exploring neighbors, explore them in sorted order from lowest to highest.

   3. 当找到这个最年轻的事务后，需要将该事务的状态设置为 abort；
   4. 当一个等待 lock 的事务状态被设置为 abort 后，还要调用请求队列中的 cv，以通知被阻塞的请求继续；
   5. 该算法需要将有向图中的所有环都 break，所以要用 while 循环来一直检测是否有环，直到没有环的存在。

## 3. Task #3 - Concurrent Query Execution

1. 任务：给 lab3 中的算子加锁，使其能够根据事务隔离级别进行并发操作。具体而言，更新 sequential scan, insert, and delete 算子中的 Next 和 Init 函数。（Task 3 不用考虑索引并发）
2. 当事务 lock/unlock 失败时，要 abort，抛出异常。
3. 此外我们还需要正确维护事务中的写集。

    > Although there is no requirement of concurrent index execution, we still need to undo all previous write operations on both table tuples and indexes appropriately on transaction abort. To achieve this, you will need to maintain the write sets in transactions, which is required by the Abort() method of transaction manager.

4. sequential scan：首先在 Init 函数中，如果隔离级别不是 read_uncommitted，则要在表上加 IS 锁；然后在 Next 函数中，如果隔离级别不是 read_uncommitted，要对该行加 S ，然后才能返回；当遍历到表末尾时，对于 read_committed，要释放之前它获取行 S 锁和表 IS 锁（注意要先释放行 S 锁，然后才能释放表 IS 锁）；而对于 repeatable，不用释放任何锁（因为 repeatable 要保证严格二阶段锁，只有在 abort 或 commit 时才释放锁）。
5. insert/delete：首先在 Init 函数中给表加 IX 锁，然后在 Next 函数中给行加 X 锁（repeatable、read_committed、read_uncommitted 对于 X 都遵守严格二阶段锁）。

## 4. 总结

lab4 需要理解不同隔离级别和不同锁类型下的加锁和解锁的过程，只要知道了这个过程就相对比较简单。此外，lab4 还需要特别注意细节，要考虑所有可能发生 abort 的情况，尽量使用 `LOG_INFO(); LOG_DEBUG();` 。
