### 锁概述
锁是计算机协调多个进程或线程并发访问某一资源的机制，应该都不陌生。​但在这之前我们先来看看并发控制，理清MVCC多版本并发控制和锁的关系，这也是之前我很迷惑的一个点

#### <span style="color: #489">并发控制技术
在数据库中，数据可以允许多个用户同时访问，因此在并发场景下需要确保数据的一致性，可以简单梳理一下，并发场景有三种：
![](https://cdn.jsdelivr.net/gh/guozizi/CDN@1.0/picture/mysql%E9%94%81-%E6%A6%82%E8%BF%B0.png)

从宽泛意义上讲，目前有**三种并发控制技术**

- 悲观并发控制(PCC)：心态悲观，假定多用户并发的事物在处理时都会引起并发冲突，每次操作数据的时候都会上锁
<span  style="color: #964036"> 先取锁再访问的策略，为数据的安全提供了保证，但是加锁会产生额外的开销，增加死锁的机会，只读型事物不会产生冲突也不需要加锁 </span>

- 乐观并发(OCC)：心态乐观，假定多用户并发的事物在处理时不会彼此互相影响，只在提交时检查有没有其它事物修改了该数据
<span  style="color: #964036"> 可以获得更大的吞吐量，但是发生冲突事物就会回滚重新执行

- 多版本并发(MVCC)：每个写操作都会创建一个新版本的数据，读操作根据可见性规则返回其中一个数据快照
<span  style="color: #964036"> 读 - 写冲突不加锁，非阻塞读的同时避免了脏读和不可重复读，但需要管理和挑选数据版本

![](https://cdn.jsdelivr.net/gh/guozizi/CDN@1.0/picture/mysql%E9%94%81-%E6%A6%82%E8%BF%B0-2.png)

对并发控制有了一定的了解，但需要**注意**：
- MVCC不是与悲观和乐观并发控制并不是对立的，很直观的一点MVCC可以在不加锁的情况下解决读-写冲突，并不能解决写-写冲突，写操作还是需要上锁
- MVCC可以与悲观并发或乐观并发结合使用来提高并发的性能
<br>

>MySQL中实现多版本两阶段锁协议，也就是MVCC+2PL（2PL是悲观并发实现的一种算法，锁只有在commit或rollback的时候释放）


####<span style="color: #489">为什么需要锁

再总结一下：
- 事物在并发场景下会发生读-读、读-写、写-写三种冲突，而冲突会导致脏读、不可重复读、幻读以及更新丢失等一些问题
- 为了保证数据的完整性和一致性，需要使用锁来支持对共享资源的并发访问，结合多版本并发控制在很多情况下避免了加锁操作
<br>
![] (https://cdn.jsdelivr.net/gh/guozizi/CDN@1.0/cat/cat4.jpg)<br>

###锁分类
MySQL中锁大致可以按照数据库的层级分为DB级别锁、表级别锁以及行级别锁，而不同的数据库引擎支持的锁类型也不同：
- MyISAM 只支持到表级锁
- InnoDB 可以支持到行级锁
<br>


![] (https://cdn.jsdelivr.net/gh/guozizi/CDN@1.0/picture/mysql%E9%94%81-%E5%88%86%E7%B1%BB.png)

### 全局锁
在DB级别对整个数据库实例加锁，加锁之后：
- 数据库处于只读状态
- 阻塞对数据的增删改以及DDL

加锁方式：lock Flush tables with read lock
释放锁：unlock tables(发生异常时会自动释放)

全局锁主要用于做全库的逻辑备份，和设置数据库只读(set global readonly=true)相比，全局锁在发生异常时会自动释放

> MyISAM、InnoDB都支持全局锁，但InnoDB一般不使用
基于InnoDB对事物的支持以及MVCC多版本并发的实现，InnoDB可以选择mysqldump工具加 –single-transaction参数，在不阻塞写操作的同时做全库的逻辑备份

<br>

### 表级别锁
表级别对操作的整张表加锁，锁定颗粒度大，资源消耗少，不会出现死锁，但并发度低，表级锁有两种模式：
- 表共享锁：对同一表的操作不阻塞读，阻塞写
- 表独占锁：对同一表的操作读写阻塞

![] (https://cdn.jsdelivr.net/gh/guozizi/CDN@1.0/picture/mysql%E9%94%81-%E8%A1%A8%E9%94%81.png)

MyISAM引擎默认支持表级别锁
表级别的锁有两种：表锁和元数据锁(MDL)

#### <span  style="color: #489">表锁
显示加锁方式：lock tables {tb_name} read/write
释放锁：unlock table {tb_name}  (连接中断也会自动释放)

MyISAM引擎下隐式加锁：
- 执行SELECT查询自动加共享锁（读锁）
- 执行INSERT、UPDATA、DELETE操作自动加独占锁（写锁）<br>
![] (https://cdn.jsdelivr.net/gh/guozizi/CDN@1.0/picture/mysql%E9%94%81-%E5%85%83%E6%95%B0%E6%8D%AE.png)<br>
>MyISAM读写锁优先级：
默认情况下写锁比读锁具有更高的优先级,即使读请求先到等待队列，写锁也会插入到读锁之前，优先执行写操作,但MyISAM也支持依据生产环境通过修改参数的设置改变读写的优先级


#### <span  style="color: #489">元数据锁（MDL）
隐式锁，主要针对对表结构改变的操作(DDL)，没有显示加锁方式，访问表时自动加锁：
- 执行DML(SELECT, INSERT...) 操作加共享锁（读锁）
- 执行DDL(ALTER, DROP...) 操作加独占锁（写锁）

![] (https://cdn.jsdelivr.net/gh/guozizi/CDN@1.0/picture/mysql%E9%94%81-%E8%A1%A8%E9%94%812.png)

到这里你是不是会有疑问：假设我要向表里增加一个字段隐式加MDL写锁，那么线上所有对这个表的增删改查(DML)操作都会阻塞
>MySQL在5.6之后引入online DDL，也就是进行DDL操作时MDL写锁会降级成读锁，线上DML操作不会被阻塞，DDL操作完成之后升级回MDL写锁然后释放

查看表级锁争用情况：SHOW STATUS LIKE 'table%' 
总之表级锁因为锁的粒度大，若一个事物执行时间过长，很可能会导致后面对这个表的请求全部阻塞

### 行级别锁
InnoDB支持行级别锁，锁粒度小并发度高，但是加锁开销大也很可能会出现死锁，锁模式：
- 共享锁(读锁) S：对同一行的操作读不阻塞，阻塞写
- 排它锁(写锁) X：对同一行的操作读写都会阻塞
- 意向共享锁 IS：一个事物想要加S锁时必须先获得该表的IS
- 意向排它锁 IX：一个事物想要加X锁时必须先获得该表的IX

![] (https://cdn.jsdelivr.net/gh/guozizi/CDN@1.0/picture/mysql%E9%94%81-%E8%A1%8C%E9%94%81.png)

>为什么需要意向锁：
意向锁是表级别的锁，用来标识该表上有数据被锁住或即将被锁，对于表级别的请求(LOCK TABLE..)，就可以直接判断是否有锁冲突，不需要逐行检查锁的状态

InnoDB的默认隔离级别RR(可重复读)，在RR下读数据有两种方式：
- 快照读：在MVCC下，事物开启执行第一个SELECT语句后会获取一个数据快照，直到事物结束读取到的数据都是一致的
<span  style="color: #964036"> 普通的 select.. 查询都是快照读
- 当前读：读取的数据的最新版本，并且在读的时候不允许其它事物修改当前记录
<span  style="color: #964036"> select.. lock in share mode(读锁)
<span  style="color: #964036"> select.. for update(写锁)

**加锁方式**：
- 普通 select... 查询 (不加锁)
- 普通 insert、update、delete... (隐式加写锁)
- select..lock in share mode (加读锁)
- select..for update  (加写锁)


**解锁**：
提交/回滚事物（commit/rollback）
kill 阻塞进程

<span  style="color: #964036"> 注：以下行级锁分析都默认RR(可重复读)的事物隔离级别

####  <span  style="color: #489">锁加在索引上
InnoDB的行锁是通过给索引上的索引项加锁来实现的

>即使在建表的时候没有指定主键，InnoDB会默认创建一个DB_ROW_ID的自增字段为表的主键，并且其主键索引（聚簇索引）为GEN_CLUST_INDEX<br>
主键索引也被称为聚簇索引

可以看下面例子，涉及到回表对聚簇索引的索引项也会加锁：
![] (https://cdn.jsdelivr.net/gh/guozizi/CDN@1.0/picture/mysql%E9%94%81-%E8%A1%8C%E9%94%812.png)

####  <span  style="color: #489"> 行级锁算法：
- Record Lock： 对对应的索引记录项加锁
- Grap Lock：对索引项之间的间隙加锁，加锁之后间隙范围内不允许插入数据，防止发生幻读
- Next-key Lock：可以理解为Record Lock+Grap Lock（InnoDB行锁默认加的是 Next-key Lock）

举个例子更好理解：
![] (https://cdn.jsdelivr.net/gh/guozizi/CDN@1.0/picture/mysql%E9%94%81-%E8%A1%8C%E9%94%813.png)

现在你可能已经知道了：
如果在加Record Lock的基础之上再加上Grap Lock问题就解决了
![] (https://cdn.jsdelivr.net/gh/guozizi/CDN@1.0/picture/mysql%E9%94%81-%E8%A1%8C%E9%94%814.png)

通过上面这个例子，我们可以看到：
- record lock  可以锁一个存在的索引项
- grap lock 锁索引项之间的间隙，可以防止幻读（左开右开区间）
- next-key lock 上面两个锁相加，innodb默认加锁单位（左开右闭区间）<br>


####  <span  style="color: #489">加锁规则
行级锁默认加 next-key lock，查询过程中访问到的索引项都会加锁，而根据不同的索引也有不同的加锁规则：
- 唯一索引等值查询：当索引项存在时，next-key lock 退化为 record lock；当索引项不存在时，默认 next-key lock，访问到不满足条件的第一个值后next-key lock退化成grap lock
- 唯一索引范围查询：默认 next-key lock，(特殊'<=' 范围查询直到访问不满足条件的第一个值为止)

- 非唯一索引等值查询：默认next-key lock ，索引项存在/不存在都是访问到不满足条件的第一个值后next-key lock退化成grap lock
- 非唯一索引范围查询：默认 next-key lock，向右访问到不满足条件的第一个值为止

<span  style="color: #964036"> 注：以上加锁规则参考《mysql 45讲》和实践验证自己总结所得，非官方规则


可能有点难理解，针对这几种情况分别举例说明一下，假设我有以下数据：
| id | name | age |
|-----|-----|------|
| 1 | 张三  | 21 |
| 4| 王一  | 26 |
| 6 | 小军  | 18 |
| 9| 小红  | 23 |

在上面的数据表我们可以得到5个next-key lock 区间：
唯一索引(id)：(-∞,1]，(1,4]，(4,6]，(6,9] ,（9,+supremum]
非唯一索引(age)：(-∞,18]，(18,21]，(21,23]，(23,26] ,（26,+supremum]

**唯一索引等值查询**：
![] (https://cdn.jsdelivr.net/gh/guozizi/CDN@1.0/picture/mysql%E9%94%81-%E8%A1%8C%E9%94%8151.png)

**唯一索引范围查询**：
![] (https://cdn.jsdelivr.net/gh/guozizi/CDN@1.0/picture/mysql%E9%94%81-%E8%A1%8C%E9%94%8161.png)

**非唯一索引等值查询**：
![] (https://cdn.jsdelivr.net/gh/guozizi/CDN@1.0/picture/mysql%E9%94%81-%E8%A1%8C%E9%94%8171.png)

**非唯一索引范围查询**：
![] (https://cdn.jsdelivr.net/gh/guozizi/CDN@1.0/picture/mysql%E9%94%81-%E8%A1%8C%E9%94%8181.png)

细心一点你会发现上面例子中：
- 唯一索引的查询用的是 select .. for update
- 非唯一索引的查询用的是 select .. lock in share model
<br>
>for update 加的是写锁，写锁默认认为会对数据做更改，不管查询有没有涉及到回表都会对聚簇索引(主键索引)加锁
lock in share model 加的是读锁，如果没有涉及到回表（像覆盖索引），不会对聚簇索引(主键索引)加锁

如果上面例子中非唯一索引的查询用的是 select .. for update，还需要分析聚簇索引(主键索引)的加锁情况

### 死锁
死锁指的是两个或两个以上的事物在执行过程中争抢锁资源而造成相互等待的情况

表锁不会出现死锁，主要还是针对InooDB的行锁，可以看下面的例子：
![] (https://cdn.jsdelivr.net/gh/guozizi/CDN@1.0/picture/mysql%E9%94%81-%E6%AD%BB%E9%94%81.png)

**监控分析锁问题**
```
# 查询InnoDB锁的整体情况
# 可以重点查看Innodb_row_lock_waits和Innodb_row_lock_time_avg这两个值
# 如果数值较大，说明锁之间的竞争大
show status like 'innodb_row_lock%';

#可以通过INNODB_TRX、INNODB_LOCKS、INNODB_LOCK_WAITS这三个表
#分析可能存在的锁的问题
select * from information_schema.INNODB_TRX; # 查看所有事物
select * from information_schema.INNODB_LOCKS; # 查看锁
select * from information_schema.INNODB_LOCK_WAITS; # 查看锁等待
```


**解决死锁**：
- 超时等待，事物超时自动回滚(innodb_lock_wait_timeout 默认50s)
- 主动死锁检测，事物请求锁的时候采用 wait-for graph 等待图的方式进行死锁检测（innodb_deadlock_detect 默认on）
- 发现死锁也可以人为 kill 进程

### 总结
- MySQL锁分为全局锁、表级锁以及行级锁，不同的存储引擎支持锁的粒度有所不同，MyISAM 只支持到表级锁，InnoDB 则可以支持到行级锁，锁的粒度决定了业务的并发度，因此更推荐使用InnoDB
- InnoDB默认最小加锁粒度为行级锁，并且锁是加在索引上，如果SQL语句未命中索引，则走聚簇索引的全表扫描，表上每条记录都会上锁，导致并发能力下降，增大死锁的概率，因此需要为表合理的添加索引，线上查询尽量命中索引
- 行级锁默认加 next-key lock，而根据不同的索引也有不同的加锁规则，我们可以根据加锁规分析加锁区间
- 锁粒度的减小提高了并发度的同时也增加了死锁的风险，查询应尽量考虑减少锁的范围
