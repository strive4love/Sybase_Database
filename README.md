# 锁 
sybase从11.9.2开始支持行锁(lock datarows)。
在数据库实现中，通过锁定机制控制数据库的并发访问，保证数据库访问的正确性。根据定义：

锁定是一种并发控制机制：它可以确保数据在同一事务中和不同事务之间保持一致。在多用户环境中，由于几个用户可能会在同一时间使用同一数据，因此需要锁定功能。

sybase锁分类
按照锁性质可以分为共享锁，排他锁。当在数据库事务中，读取信息时，会对数据库添加共享锁。当修改信息时，会添加排他锁。

按照锁的粒度，可以分为行锁，页锁，表锁等。

sybase隔离级别
sybase分为0,1,2,3四个隔离级别。

0  读取未提交的，允许事务读取未提交的数据更改，排他锁在对数据库进行写操作后立即释放，不会持有到事务提交或回滚。 
1 读取已提交的，仅允许事务读取已提交的数据更改，排他锁持有到事务提交或回滚，但共享锁在加载数据到内存后立即释放。 
2 可重复读取事务可重复同一查询，事务读取过的任何行都不会被更新或删除，排它锁和共享锁都会持有到事务结束，查询结果集不可以删除和修改，但是可以插入。 
3 可串行化读取事务可重复同一查询，且得到完全相同的结果。不能插入任何将出现在结果集中的行，排它锁和共享锁都会持有到事务结束，查询结果集不可以增删改。

可以使用select @@isolation语句查看数据库的隔离级别。

sybase数据库通常隔离界别设置为1，值得注意的是使用WAS通过jdbc连接数据库上，经常会将隔离级别提升为2。在使用E-SQL编译时，通常将隔离级别提升为3。

sybase死锁
sybase数据库出现死锁，即处于死锁中的各事务，都持有锁，但又都在等待其他锁，从而组成一个环，造成死锁。

最简单的死锁情况，事务T1，T2，执行顺序相反，会造成死锁，情形如下：

执行顺序

T1	T2
1	排他锁A	 
2	 	排他锁B
3	排他锁B	 
4	 	排他锁A
这时候，会出现事务T1持有排他锁A，同时等待排他锁B，事务T2持有排他锁B，等待排他锁A。这是就造成了T1等待T2释放排他锁B，T2等待T1释放排他锁A，形成一个死锁环。

多个事务出现死锁时，情形与两个事务死锁相比，只是环更大了一些，环上的节点多了一些。其本质仍然是形成一个等待环。

隔离级别对死锁的影响
隔离级别同样会对锁定有很大的影响，例如，

情形一、

执行顺序

T1	T2
1	排他锁A	 
2	 	排他锁B
3	共享锁B	 
4	 	共享锁A
当隔离级别为0时，不会出现死锁。当隔离界别为1,2,3时，则会发生死锁。

情形二、

执行顺序

T1	T2
1	共享锁A	 
2	 	共享锁B
3	排他锁B	 
4	 	排他锁A
当隔离级别为0,1时，不会出现死锁。当隔离界别为2,3时，会发生死锁。

情形三、

该情况是最近在系统中发现的一个死锁问题。程序从文件导入数据到数据库中，每次导入一条记录时，首先尝试以update的方式导入一条记录，当找到记录为空时，则将该条记录更改为以insert的方式导入到数据库中。

同时，导入过程是由多个进程共同完成的，每个进程导入一个文件，多个进程同时工作，然而当程序运行时，多个进程同时导入出现死锁。

通过监控sybase日志，发现死锁都是发生在insert时，出现next-key lock。sybase日志保存在安装目录下，例如安装目录为/sybase/ASE-12_5，日志文件为/sybase/ASE-12_5/install/db_name.log。

通过检查数据库的隔离级别，为1，没有发现异常，百思不得其解。

后在程序中添加查询数据库隔离级别语句，以检查在程序运行中到底隔离级别是多少？

经检查，隔离级别为3，也就是说在事务中，不能插入任何将出现在结果集中的行，下面分析一下出现死锁的原因。

当两个进程同时插入记录到同一个间隙中时，每个事务可能由两个操作组成

1.update

2.insert，当update结果集为空时，则转为insert。

其执行过程中，两个进程可能出现以下运行情况

执行顺序

T1	T2
1	共享锁A	 
2	 	共享锁B
3	排他锁B	 
4	 	排他锁A
例如，目前数据库只有一条记录，主键为5，此时T1，T2分别插入主键为3,4的数据，由于两个事务都在运行之中，因此T1，T2都会尝试在5之前插入数据，首先其在update时，会产生共享锁，由于隔离级别为3，此时两个事务尝试插入时都会失败，要解决这种死锁，可以在程序中显式设置隔离级别为1。

sybase锁升级
sybase同时提供锁升级的功能，例如将行锁升级为页锁，将页锁升级为行锁。具体参数可以进行设置。

例如当某一页中90%的行都被锁定，那么此时sybase可能将这些多个行锁升级为一个页锁，锁定整个页。这也是造成死锁一个重要的原因。

有时，根据判断，不会产生死锁。

执行顺序

T1	T2
1	行级排他锁A	 
2	 	行级排他锁B
3	行级排他锁C	 
4	 	行级排他锁D
在上述情况中，如果没有锁升级机制，是无论如何也不会产生死锁的。但是当有了锁升级机制之后，可能T1在将行级锁A升级为页锁Pa，T2将行锁B升级为页锁Pb，而T1需要访问的行C在页Pb中，T2需要访问的D在也Pa中，这时就会构成一个锁定环，构成死锁。

 

总结
在软件系统实现中，经常会采用数据库。使用数据库时死锁问题是大家通常都会遇到的，遇到死锁首先需要分析原因，定位问题，这也是最关键的一步。

出现死锁的原因很多，但本质一定是多个进程出现了相互等待，而每个进程又都持有锁，从而形成了一个依赖关系环，因此问题最关键的就是找到这个依赖关系，以及是哪几个事务，哪几个锁导致了死锁，只要确定了这几点，死锁问题将迎刃而解。

#  聚合索引(clustered index) / 非聚合索引(nonclustered index)
微软的SQL SERVER提供了两种索引：聚集索引（clustered index，也称聚类索引、簇集索引）和非聚集索引（nonclustered index，也称非聚类索引、非簇集索引）。正文内容本身就是一种按照一定规则排列的目录称为"聚集索引"，正文部分本身就是一个目录，您不需要再去查其他目录来找到您需要找的内容。每个表只能有一个聚集索引 ，因为目录只能按照一种方法进行排序。
我们把这种目录纯粹是目录，正文纯粹是正文的排序方式称为"非聚集索引"

何时使用聚集索引或非聚集索引      

下面的表总结了何时使用聚集索引或非聚集索引（很重要）。        
   

动作描述                        使用聚集索引                       使用非聚集索引     
   

列经常被分组排序              应                                       应     

返回某范围内的数据          应                                       不应        

一个或极少不同值           不应                                     不应     

小数目的不同值                应                                       不应     

大数目的不同值              不应                                      应     

频繁更新的列                 不应                                      应     

外键列                            应                                        应     

主键列                            应                                        应     

频繁修改索引列             不应                                       应     
    
    
# Client-Library

# bcp
Transact-SQL commands cannot transfer data in bulk. For this reason, use bcp for any large transfers. 
bcp as a standalone program from the operating system can read or write files in a wide variety of formats. When copying in from a file, bcp inserts data into an existing database table; when copying out to a file, bcp overwrites any previous contents of the file. Versions earlier than 15.0.3 did not allow you to run fast bcp on tables with nonclustered indexes or triggers. Cluster Edition version 15.0.3 and later removes this restriction.
The utility is located in:
(UNIX) $SYBASE/$SYBASE_OCS/bin/bcp  
(UNIX) $SYBASE/$SYBASE_OCS/bin/bcp_r(for parallel bcp)  
(Windows) %SYBASE%\%SYBASE_OCS%\bin\bcp.exe 

# bcp Modes
bcp in works in one of three modes.
Slow bcp – logs each row insert that it makes, used for tables that have one or more indexes.
Fast bcp – logs only page allocations, copying data into tables without indexes or at the fastest speed possible. Use fast bcp on tables with nonclustered indexes.
Fully logged fast bcp – provides a full log for each row. Allows you to use fast bcp on indexed and replicated tables.
Although fast bcp might enhance performance, slow bcp gives you greater data recoverability. Fully-logged fast bcp provides a combination of both.

Table Properties |                                 bcp Mode for bulkcopy on, With Logging  |  bcp Mode for bulkcopy on Without Logging
 
Clustered index  |                                                              Slow mode  |  Slow mode
 
Replicated table with indexes or triggers, but no clustered index |             Fast mode  |  Slow mode
 
Nonclustered index and no triggers |                                            Fast mode  |  Slow mode
 
Triggers, and no indexes           |                                            Fast mode  |  Fast mode
 
Nonclustered index with triggers   |                                            Fast mode  |  Fast mode
 
No indexes, no triggers, and no replication  |                                  Fast mode  |  Fast mode
 

# parallel copy and locks
Starting many current parallel bcp sessions may cause Adaptive Server to run out of locks.
When you copy in to a table, bcp acquires an exclusive intent lock on the table, and either page or row locks, depending on the locking scheme. If you are copying in very large tables, and especially if you are performing simultaneous copies into a partitioned table, this can require a very large number of locks.To avoid running out of locks: Set the number of locks configuration parameter high enough, or Use the -b batchsize bcp flag to copy smaller batches. If you do not use the -b flag, the entire copy operation is treated as a single batch.

# Fast, Fast-logged, and Slow bcp
MXG_BODEV1              	  420000.0 MB	sa                      	4	Jun 22, 2013      	select into/bulkcopy/pllsort, trunc log on chkpt, abort tran on log full
MXG_BODEV2              	  420000.0 MB	sa                      	5	Jun 22, 2013      	select into/bulkcopy/pllsort, trunc log on chkpt, abort tran on log full
MXG_BODEV3              	  420000.0 MB	sa                      	6	Jul 08, 2013      	select into/bulkcopy/pllsort, trunc log on chkpt, abort tran on log full
MXG_BODEV5              	  321200.0 MB	sa                      	8	Jun 23, 2013      	select into/bulkcopy/pllsort, trunc log on chkpt, abort tran on log full
MXG_BODEV8              	  420000.0 MB	sa                      	10	Jun 23, 2013      	select into/bulkcopy/pllsort, trunc log on chkpt, abort tran on log full
MXG_ENV906              	  420000.0 MB	sa                      	14	May 31, 2016      	select into/bulkcopy/pllsort, trunc log on chkpt, abort tran on log full
MXG_IRDFX_DEV9          	  420000.0 MB	sa                      	13	May 25, 2016      	no options set ！！！！！
MXG_PRC                 	  420000.0 MB	sa                      	9	Jun 23, 2013      	select into/bulkcopy/pllsort, trunc log on chkpt, no chkpt on recovery, abort tran on log full
MXG_SABREPV             	  420000.0 MB	sa                      	7	Jun 23, 2013      	select into/bulkcopy/pllsort, trunc log on chkpt, abort tran on log full
MXML_BODEV1             	   91192.0 MB	sa                      	12	Jun 24, 2013      	select into/bulkcopy/pllsort, trunc log on chkpt, abort tran on log full
MXML_BODEV2             	   91192.0 MB	sa                      	11	Jun 24, 2013      	select into/bulkcopy/pllsort, trunc log on chkpt, abort tran on log full
master                  	      50.0 MB	sa                      	1	Jun 19, 2013      	mixed log and data
model                   	       4.0 MB	sa                      	3	Jun 19, 2013      	mixed log and data
sybsystemdb             	      54.0 MB	sa                      	31513	Jun 19, 2013      	mixed log and data
sybsystemprocs          	     200.0 MB	sa                      	31514	Jun 19, 2013      	trunc log on chkpt, mixed log and data
tempdb                  	   20004.0 MB	sa                      	2	Mar 24, 2016      	select into/bulkcopy/pllsort, trunc log on chkpt, mixed log and data
