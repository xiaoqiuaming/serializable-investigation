# serializable-investigation
## 基础锁设施
SIREAD lock有三种粒度：tuple、page、relation。如果SIREAD lock锁住了一个页的多个元组，那么就会聚合成一个页的锁，并且释放元组的锁。同理，页级锁也可以升级成表锁。在开始对元组加siread lock的时候，对页也加上siread lock，开始对表做顺序扫描的时候也需要对表加上siread lock而不关心索引条件，这个会造成一些false-positive。

每次在读操作的时候创建siread lock，并且调用CheckForSerializableConflictOut，建立rw冲突关系。UDI操作的时候需要调用CheckForSerializableConflictIn检查siread lock建立rw冲突关系。在提交的时候调用PreCommit_CheckForSerializationFailure，如果发现危险结构则终止，遵循First committer win原则。

两阶段提交相关：AtPrepare_PredicateLocks、PostPrepare_PredicateLocks、PredicateLockTwoPhaseFinish、predicatelock_twophase_recover。

Pg是多进程模型，SIREAD lock存储在mmap共享存储区，每个backend都可以访问
![text](https://github.com/xiaoqiuaming/serializable-investigation/blob/main/%E5%9B%BE%E7%89%871.png)
![text](https://github.com/xiaoqiuaming/serializable-investigation/blob/main/%E5%9B%BE%E7%89%872.png)


PredicateLockTargetHash存储lock table 里的所有PREDICATELOCKTARGET，PREDICATELOCKTARGET是锁对象描述符+SIREAD lock列表。PredicateLockHash存储了所有的SIREAD lock。锁对象描述符是4个32比特的id值，分别表示db id、relation id、block id、offset。举个例子，如果锁粒度是block，page，那么锁对象描述符的offset就设置为InvalidOffsetNumber。FinishedSerializableTransactions组织所有完成提交的但是还未清理siread lock的事务。一个SIREAD LOCK用一个PREDICATELOCKTARGET+所属的事务指针标识，连接到PREDICATELOCKTARGET的SIREAD lock队列和事务的predicateLocks队列。

表示rw冲突的数据结构是RWConflictData，对于每对有rw冲突的事务，都会用这个结构关联。
![text](https://github.com/xiaoqiuaming/serializable-investigation/blob/main/%E5%9B%BE%E7%89%873.png)

对于每个事务来说，会有多个左端rw冲突、右端rw冲突，有多个siread lock锁。
![text](https://github.com/xiaoqiuaming/serializable-investigation/blob/main/%E5%9B%BE%E7%89%874.png)

SIREAD lock的设置，分为三种粒度，分别是tuple、page、relation。
![text](https://github.com/xiaoqiuaming/serializable-investigation/blob/main/%E5%9B%BE%E7%89%875.png)

每次更新数据的时候就会检查：1、是不是已经存在siread lock；2、是不是被更高粒度的锁cover。如果是这两种情况就不需要额外加siread lock。
![text](https://github.com/xiaoqiuaming/serializable-investigation/blob/main/%E5%9B%BE%E7%89%876.png)

创建SIREAD LOCK，先得到锁target（没有则创建），再创建siread lock连接到target的SIREAD lock队列和事务的predicateLocks队列。

合并细粒度锁到粗粒度。建完siread lock之后增加父级对象的孩子锁数量，如果大于某个阈值，就升级为父级对象siread lock，释放所有孩子锁。
![text](https://github.com/xiaoqiuaming/serializable-investigation/blob/main/%E5%9B%BE%E7%89%877.png)

## 清理SIREAD LOCK
  在事务每次提交的时候，尝试清理过期的siread lock。清理的条件是事务发现自己的xmin（pg中的snapshot）是活跃事务中的最小xmin，并且是最后一个，那么就更新global xmin。
  ![text](https://github.com/xiaoqiuaming/serializable-investigation/blob/main/%E5%9B%BE%E7%89%878.png)

  清理过期的siread lock。遍历所有FinishedSerializableTransactions列表里的事务，对所有xmax小于global xmin的事务，释放siread lock（从PredicateLockHash中删除这个锁，从PredicateLockTargetHash对应的lock队列中删除这个锁）同时清除所有inConflicts（左端rw冲突）。如果是只读事务，那么同时清除右端的rw冲突。

## 冲突检测。
读的时候根据查询范围创建对应粒度的锁。读、写、提交时标记rw冲突，并终止危险结构中的事务。

writer写的时候，需要检查三种粒度的rw冲突。检测的时候得到锁对象的所有siread lock，通过siread lock得到对应的reader事务，对每个reader检测危险结构。在reader和writer之间建立rw冲突
![text](https://github.com/xiaoqiuaming/serializable-investigation/blob/main/%E5%9B%BE%E7%89%879.png)

当读取一行数据的旧版本的时候，那么建立rw冲突。读写的时候只检测、设置冲突

读写设置冲突时，都会检测串行化冲突。
![text](https://github.com/xiaoqiuaming/serializable-investigation/blob/main/%E5%9B%BE%E7%89%8710.png)

几种情况：

1、W是pivot事务并且W、T2已经提交，T2比W提交早，终止事务

![text](https://github.com/xiaoqiuaming/serializable-investigation/blob/main/%E5%9B%BE%E7%89%8711.png)

2、W是pivot事务，T2比R、W提交早终止本事务。如果reader已经提交、writer已经提交、reader是read only并且在T2提交之前获取的snapshot，就不用终止事务。

![text](https://github.com/xiaoqiuaming/serializable-investigation/blob/main/%E5%9B%BE%E7%89%8712.png)

3、R是pivot事务，并且writer已经prepare，T0的提交时间戳比writer prepare version大，那么需要终止。
![text](https://github.com/xiaoqiuaming/serializable-investigation/blob/main/%E5%9B%BE%E7%89%8713.png)

4、判断需要终止事务，就判断writer是不是自己，是的话就ereport终止自己，不是的话判断writer是不是已经prepare，已经prepare就终止自己，不是的话就标记writer为doomed。

提交的时候再做最后的check，判断自己前面两个rw冲突都没提交的话，就优先终止pivot事务，如果pivot事务已经prepare，就自杀。
