### 1.锁的分类（一次封锁或两段锁）

a.一次封锁

就是在方法的开始阶段，已经预先知道要用到那些数据，在方法开始之前就将这些数据用锁锁住，在方法执行完成之后，再全部解锁。这种方式可以有效的避免死锁，但是这种封锁方式在数据库中并不可用，因为在事务开始阶段，并不会提前预知要使用数据库中的哪些数据。因此数据库遵循两段锁协议。

b. 两段锁

就是将事务分为加锁阶段和解锁阶段两个阶段。

加锁阶断：在这个阶段，可以进行加锁操作。在对事务进行"读"的操作之前，需要申请S锁（共享锁），对事务进行"写"的操作之前，需要申请X锁（排它锁）。在这里要注意的是，如果事务加上S锁，其他事务可以继续加共享锁，但不能加排它锁；但是如果事务加上X锁，其他事务则不能加其他任何类型的锁。如果事务加锁不成功，则事务进入wait状态，直至枷锁成功为止。
   
解锁阶断：当事务提交之后，就会释放锁，此时事务进入解锁阶断，在这个阶段只能进行解锁操作，不能再进行枷锁操作。
    两段锁无法避免死锁，但是却可以保证事务的并发操作是串行化进行的。



### 2.Mysql中锁的种类

a.表锁：对整张表进行加锁

b.行锁：锁住数据行，对有限的数据行进行加锁，对其他数据不限制。

在这里要注意的是如果查询条件中存在没有建索引的字段，Mysql则会给数据库中整张表的所有数据行加行锁，这时存储引擎会将所有的数据记录加锁返回给MysqlServer层进行过滤，在MysqlServer层过滤的时候，如果发现有不满足过滤条件的记录，则会调用unlock_row方法，将不满足条件的记录所加的锁释放掉，当然这种做法是违背了"两段锁协议"的。



### 3.事务的隔离级别

| 隔离级别                  | 脏读    | 不可重复读  | 幻读 |
|----------------------------|--------|------------|----- |
|读未提交（ReadUncommitted） | 可能   | 可能       | 可能 |
|读已提交(ReadCommitted)     | 不可能 | 可能       | 可能 |
|可重复读(Repeateable Read)  | 不可能 | 不可能     | 可能 |
|可串行化（Serializable）    | 不可能 | 不可能     |不可能|

>a.读未提交（RU）：就是可能会读取到其他会话中未提交事务的数据。
RU级别中，无论是读的操作还是写的操作，都不会进行加锁操作

>b.读已提交（RC）：只能读到已经提交事务的数据。Oracle默认的隔离级别为该级别。
RC级别中，数据的读取操作不会进行加锁，但是数据的写入、修改和删除操作是需要加锁的。

>c.可重复读（RR）：指同一个事务的多个实例在并发读取数据时看到的数据行是一致的。InnoDB引擎默认为该级别。
RR级别中， 数据的读取操作不会进行加锁，但是数据的写入、修改和删除操作是需要加锁的。

>d.可串行化：每次读取的时候事务都要获取表级别的S锁，每次写入的时候都会加写锁（X锁），读写操作会相互阻塞。



### 4.不可重复读和幻读的区别

不可重复读是指如果事务A读取了某一行数据，事务A未提交，如果事务B修改了这一行数据并且提交，事务A再去重复读取这条记录时，会发现两次读取的同一行数据记录发生了变化。

幻读是指事务A读取了某一个范围内的数据，事务A未提交，事务B在这个范围内插入一条记录，事务A再去重复读取这个范围内的数据时，发现再次读取到的数据记录数比之前多。

不可重复读的重点在于update和delete操作，而幻读的重点在于insert操作。就拿上面的例子来说，如果只是纯粹的用锁机制来实现这两种隔离级别的话，在sql执行事务A的select操作之后，就对这些数据进行加锁，在此期间，其他事务拿不到该行锁，因此对于事务B中的update操作，只能等待事务A将锁释放之后才能对相应的数据行加锁。这样就可以实现可重复读，但是这种方法却无法锁住即将要insert的数据，因此就会出现"幻读"的现象。

因此，"幻读"现象是不能通过行锁来避免的，必须通过Serializable隔离级别来避免，也就是读的操作用"读锁"，写的操作用"写锁"，"读锁"和"写锁"互为互斥锁，这么做就可以有效地避免"脏读"、"不可重复读"、"幻读"等问题，但是会极大地降低数据库的并发能力。

前面所说的都是使用悲观锁来解决"不可重复读"和"幻读" 这两个问题的，但是Mysql、Oracle、Postgresql等一些成熟的关系型数据库为了性能上的考虑，都会采用以乐观锁为基础的多版本并发控制（MVCC）来避免这两个问题。

### 5.悲观锁和乐观锁

a.悲观锁：在整个数据的处理过程中，将数据处于锁定的状态，悲观锁的实现，是依靠数据库提供的锁机制，以保证操作上最的程度的独占性。在悲观锁的情况下，为了保证事务的隔离性，就需要一致性锁定读，在读取数据时对数据加锁使得其他事务无法对其进行修改操作；在修改数据时，也要对其进行加锁操作，使得其他事务无法读取这些数据。这样随之而来的就是数据库性能上的大量开销，尤其是对于长事务而言，这种开销是无法承受的。


b.乐观锁：基于数据库版本记录机制来实现的，也就是说为数据增加一个版本控制的标识。在基于数据库表的版本控制实现机制中，一般是通过为数据库表增加一个"version"字段，在读取时，将此版本号一起读出，在对数据进行修改操作时，"version"字段的值加1。在提交数据时，将要提交数据的版本号和数据库表中对应数据的version字段的值进行比较，如果要提交的数据版本号大于当前数据库表中的当前版本号，则对数据予以更新；否则，就认为要提交的数据为过期数据。


### 6.InnoDB中乐观锁的实现

在InnoDB中，会在每行数据后面添加两个隐藏的属性来实现多版本并发控制。一个属性记录这行数据的创建时间，另外一个属性记录当前数据的过期时间（被删除时间）。但在实际的操作中，存储的值并非是时间，而是事务的版本号，每开启一个新的事务，事务的版本号就会递增。RR事务隔离级别下，

- select：读取出来的创建的版本号<=当前事务的版本号，以确保事务读取的行要么是在事务开始之前已经存在的，要么是事务自身插入或者修改过的；而删除的版本号为空或>当前事务的版本号，以保证事务所读取到的行在事务开始之前未被删除。
- insert：保存当前事务的版本号为行的创建版本号。
- update：插入一条新纪录，保存当前事务的版本号为行的创建版本号，同时保存当前事务的版本号到原来删除的行。
- delete：保存当前事务的版本号为行的删除版本号。

通过MVCC，虽然每行记录都需要额外的存储空间，更多的行检查工作以及一些额外的维护工作，但可以减少锁的使用，大多数读操作都不用加锁，读数据操作很简单，性能很好，并且也能保证只会读取到符合标准的行，也只锁住必要行。在这里要注意的是，MVCC只在RC和RR两种隔离级别下工作。

> 参考资料
- [InnoDB中事务隔离级别和锁的关系](http://asialee.iteye.com/blog/2353254)