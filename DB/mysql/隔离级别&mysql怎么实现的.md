# 在并发执行事务时会发生什么问题呢？

1. **脏读**：一个事务读到另一个事务未提交的更新数据（事务A和B并发执行，B事务执行更新后，A事务查询B事务没有提交的数据，B事务回滚，则A事务得到的数据不是数据库中的真实数据。也就是脏数据，即和数据库中不一致的数据）。
2. **不可重复读**：一是指在一个事务内，多次读同一数据。在这个事务还没有结束时，另外一个事务也访问该同一数据。那么，在第一个事务中的两次读数据之间，由于第二个事务的修改，那么第一个事务两次读到的的数据可能是不一样的。这样在一个事务内两次读到的数据是不一样的，因此称为是不可重复读。
3. **覆盖更新**：这是不可重复读中的特例，一个事务覆盖另一个事务已提交的更新数据（即A事务更新数据，然后B事务更新该数据，A事务查询发现自己更新的数据变了）。
4. **幻读**：是指当事务不是独立执行时发生的一种现象，例如第一个事务对一个表中的数据进行了修改，这种修改涉及到表中的全部数据行，同时，第二个事务也修改这个表中的数据，这种修改是向表中插入一行新数据。那么，以后就会发生操作第一个事务的用户发现表中还有没有修改的数据行，就好象发生了幻觉一样。

# 隔离级别

* Serializable (序列化，串行化) : 串行执行 **可避免脏读，不可重复读，幻读的发生。**它是最高的事务隔离级别，同事花费的代价也是很大的，性能很低，一般很少使用。

* Repeatable read (重复读) :一个事务开始读取这条数据，那么别的事务就不能对其进行修改。 可以解决不可重复读和脏读

* Read committed (读已提交): 一个事物要等另一个事务提交之后才能读取 可避免脏读 

* Read uncommitted(读未提交) 

  **mysql 默认是 `REPEATABLE-READ` 重复读**

# 如可查看级别

* 查看当前会话隔离级别

  `select @@tx_isolation;`

* 查看系统当前隔离级别

  `select @@global.tx_isolation;`

* 设置当前会话隔离级别

  `set session transaction isolatin level repeatable read;`

* 设置系统当前隔离级别

  `set global transaction isolation level repeatable read;`

# 四个隔离界别如何实现的

## Read uncommitted(读未提交) 

不管要查询的行的数据是否以提交，如果在内存中，那么用该数据按照 [MVCC-查询规则](#MVCC-查询规则)查询

如果内存中没有，那么去读磁盘数据，按照 [MVCC-查询规则](#MVCC-查询规则)查询

## Read committed (读已提交)

RC级别: read view 会在每一个 select语句执行的时候都会去重新拉一次数据，这决意味着每一次查询数据使用用来判断得read view都是最新的，保证了不会读到未提交的数据

如果当前数据在内存中，那么以内存中的数据作为当前数据，如果不存在去磁盘查询数据作为当前数据，然后使用该数据中存放的事务id ,将其作为[read_view_sees_trx_id函数](#read_view_sees_trx_id) 的trx_id参数，判断是否可以使用该数据按照 [MVCC-查询规则](#MVCC-查询规则)查询，如果不可以，则用当前数据的回滚指针指向的undo log数据继续上面的判断

## Repeatable read (重复读)

RR级别：read view 数据是在事务一开始就创建只创建一次

### 读事务早于写事务的情况

这时候读事务 read view 数据中不会涉及到改行未提交数据，如果当前数据在内存中，那么以内存中的数据作为当前数据，如果不存在去磁盘查询数据作为当前数据，然后使用该数据中存放的事务id ,将其作为[read_view_sees_trx_id函数](#read_view_sees_trx_id) 的trx_id参数，这时候一定会返回true,**此时读事务的版本小于写事务**，不管当前数据是否提交，只要按照[MVCC-查询规则](#MVCC-查询规则)查询，那么读事务在任何时候查询到的数据都是一样的。

### 读事务晚于写事务的情况

如果写事务已提交 ，这种情况就不看了，肯定都是读取到最新数据

如果未提交的时候 读事务开始（不过重复读过程中写事务是否提交了。因为热爱的view 之构建一次，这样才能保证这种情况还能满足当前事务级别，不然可能读到已提交的数据，导致不满足该事务级别了） 这时候读事务 read view 数据中就存在该行未提交数据，此时该数据肯定在内存中，然后使用该数据中存放的事务id ,将其作为[read_view_sees_trx_id函数](#read_view_sees_trx_id) 的trx_id参数，这时候一定会返回false, 这时就要使用当前数据的回滚指针指向的undo log数据继续上面的判断，最后按照[MVCC-查询规则](#MVCC-查询规则)查询

## Serializable 界别

运用锁机制来达到完全同步

> 更多关于MVCC的请看[MVCC](MVCC.md)

## MVCC-查询规则

检验该行记录的DB_TRX_ID数据行版本号`<=`当前事务版本号`，如果小于就直接返回数据，大于就根据 `DB_ROLL_PT 回滚指针`找到上一个版本数据（undo log）,继续判断版本，直到找到数据版本号<=当前事务版本号的数据返回 , 这样也就确保了读取到的数据是当前事务开始前已经存在的数据，或者是自身事务改变过的数据

## read_view_sees_trx_id

read view 中存放了当前未commit得活跃的事务id列表
`[up_limit_id，low_limit_id ]`

**1. 当行记录的事务ID小于当前的最小活动id，就是可见的。**
　　if (trx_id < view->up_limit_id) {
　　　　return(TRUE);
　　}
**2. 当行记录的事务ID大于当前的最大活动id，就是不可见的。**
　　if (trx_id >= view->low_limit_id) {
　　　　return(FALSE);
　　}
**3. 行记录的事务ID在活动范围之中时，判断是否在活动链表中，如果在那就是未commit就不可见，如果不在就是可见的。**
　　for (i = 0; i < n_ids; i++) {
　　　 view_trx_id
　　　　　　= read_view_get_nth_trx_id(view, n_ids - i - 1);
　　　　if (trx_id <= view_trx_id) {
　　　　return(trx_id != view_trx_id);
　　　　}
　　}

****