# MVCC版本号
https://www.jianshu.com/p/5d500d76e719

## MVCC版本号到底是什么？

## 第一种解释：系统版本号
- InnoDB维护一个自增的系统版本号。每开始一个新的事务，系统版本号会递增。事务开始时的系统版本号作为事务的版本号。
- INSERT：创建版本号是当时的系统版本号，不是当前事务的版本号。
- DELETE：删除版本号是当时的系统版本号，不是当前事务的版本号。
- UPDATE：旧记录的删除版本号是当时的系统版本号，新记录的创建版本号也是当时的系统版本号，不是当前事务的版本号。

第一个SELECT执行的时候，当前事务取到了系统版本号n，系统版本号自增为n+1。此后，其他事务的更新操作能取到的系统版本号最小为n+1，所以当前事务再次SELECT将看不见它们的更新。
> 但是，当前事务的更新操作怎么可以看到？它的创建/删除版本号难道不也是n+1吗？这里是特殊处理的吗？

## 第二种解释：事务标识符
添加三个隐藏字段：DB_TRX_ID，DB_ROLL_PTR，DB_ROW_ID
|字段|说明|
|------|------|
|DB_ROW_ID|插入新记录的行ID|
|DB_TRX_ID|事务标识符|
|DB_ROLL_PTR|指向undo log|

- 第一个SELECT执行时，会创建一个read view，也就是trx_sys的快照，记录了当时活跃的读写事务列表（不包括只读事务） 
- 再次SELECT的时候，通过read view来判断某一行记录是否可以被当前事务看到。如果可以就输出，不可以就利用undo log构建历史版本，再继续判断
- 先检查DB_TRX_ID标记的事务是否是当前事务，如果是就输出(?)
- 然后检查事务是否在read view里面，如果在就通过undo log返回到上一个版本，然后继续，直到事务不在read view里才输出

## InnoDB幻读问题
https://www.jianshu.com/p/e434cf370daa

## InnoDB可重复读能不能解决幻读问题？

## 第一种解释：不能
事务A一开始select出3条记录，ID分别是1/2/3，然后尝试插入ID为4的记录。此时正好事务B已经插入了ID为4的记录，并且提交。那么事务A的插入操作会失败，因为事务A已经读了事务B插入的ID为4的记录。这就是幻读。

## 第二种解析：能
InnoDB会加Netx-Key Lock，包括行锁(Record Lock)和间隙锁(Gap Lock)。间隙锁会锁住后面没有的记录，可以用来解决幻读的问题。比如事务A一开始使用select ... for update独出3条记录，此时由于间隙锁的存在，事务B将不能插入ID为4的记录，所以就不存在幻读的问题。
