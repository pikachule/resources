# MySQL篇 

## 基础概念
> 单进程多线程的数据库实例，~~数据库连接~~->数据库实例连接，MySQL中建立一个会话，不是和具体的数据库相连接，而是跟某个instance建立会话（每个会话可以使用不同的用户身份）。
> 而一个实例可以操作多个数据库，故一个会话（在操作系统概念里，会话即是线程）可以操作一个实例上的多个数据库。
> [进程与线程的一个简单解释](https://github.com/pikachule/resources/wiki/%E8%BF%9B%E7%A8%8B%E4%B8%8E%E7%BA%BF%E7%A8%8B%E7%9A%84%E4%B8%80%E4%B8%AA%E7%AE%80%E5%8D%95%E8%A7%A3%E9%87%8A)
> 权限校验(账号、密码、IP)，TCP/IP
## 基础操作
### 配置文件 my.cnf 
- Windows .ini
- Linux /etc/my.cnf
```
mysql --help | grep my.cnf
```
```
                      order of preference, my.cnf, $MYSQL_TCP_PORT,
/etc/my.cnf /etc/mysql/my.cnf /usr/local/mysql/etc/my.cnf ~/.my.cnf 

```
> 配置读取顺序为：/etc/my.cnf -> /etc/mysql/my.cnf -> /usr/local/mysql/etc/my.cnf -> ~/.my.cnf ,同一参数以最后一个配置文件为准。

### 系统变量(系统参数) show variables;
> eg. 数据库路径 datadir
```
show variables like 'datadir';
```
| Variable_name | Value        |
|---------------|--------------:|
| datadir       | /data/mysql/ |

> eg. 版本号
```
show variables like 'version%';
```

| Variable_name           | Value                        |
| ------------- | -----:|
| version                 | 5.6.38-log                   |
| version_comment         | MySQL Community Server (GPL) |
| version_compile_machine | i686                         |
| version_compile_os      | linux-glibc2.12              |
> eg. 线程
```
show processlist;
show full processlist;
```
|Status||
| --- | ---:|
|Sending data | The thread is reading and processing rows for a SELECT statement, and sending data to the client. Because operations occurring during this state tend to perform large amounts of disk access (reads), it is often the longest-running state over the lifetime of a given query.  |

详见：[https://dev.mysql.com/doc/refman/5.7/en/general-thread-states.html](https://dev.mysql.com/doc/refman/5.7/en/general-thread-states.html)
```
show status;
```
[https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html](https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html)



### mysqladmin
> eg. 状态
```
mysqladmin status -uroot -p

Uptime: 4661  Threads: 1  Questions: 200  Slow queries: 0  Opens: 16  Flush
tables: 1  Open tables: 6  Queries per second avg: 0.043
```
详见：[https://github.com/pikachule/resources/wiki/15-Practical-Usages-of-Mysqladmin-Command-For-Administering-MySQL-Server](https://github.com/pikachule/resources/wiki/15-Practical-Usages-of-Mysqladmin-Command-For-Administering-MySQL-Server)

## Mysql体系结构
![](http://7xvulr.com1.z0.glb.clouddn.com/540235-20160927083854563-2139392246.jpg)
> 组成部分
- 连接池组件
- 管理服务和工具组件
- SQL接口组件
- 查询分析器组件
- 优化器组件
- 缓冲(Cache)组件
- 插件式存储引擎 (基于表)
- 物理文件 (二进制文件，通过数据库实例进行操作)

## InnoDB

> Heikki Tuuri，v5.5+默认引擎，行锁、ACID[原子性Atomicity、一致性Consistency、隔离性Isolation、持久性Durability]事务(OLTP)、~~支持外键~~、单.ibd文件(.frm)，clustered聚合按主键顺序存放(没有显式定义主键时每行会生成一个6字节的ROWID作为主键)，

> InnoDB存储引擎有多个内存块，可以认为这些内存块组成了一个大的内存池，负责：
- 维护所有进程/线程需要访问的多个内部数据结构
- 缓存磁盘上的数据
- 重做日志的缓冲（redo log）

### 后台线程
> 后台线程的主要作用是负责刷新内存池中的数据，保证缓冲池内的数据是最近的数据；同时将脏数据刷回磁盘；保证数据库发生异常时InnoDB的恢复。

> InnoDB是一个多线程模型，每个线程负责不同的任务。

- Master Thread //主线程，主要负责将缓冲池中的数据异步刷新到磁盘，保证数据一致性(脏页的刷新、Insert Buffer、Undo页回收)
- IO Thread //Async IO，负责IO请求回调，
```
show engine innodb status;
```

- Purge Thread //回收事务commit后的undolog，1.1独立于主线程，1.2支持多线程

```
show variables like 'innodb_purge_threads';
```

- Page Cleaner Thread //脏页刷新


### 内存

- 缓冲池 
//命中直接读取，否则磁盘读取；对数据库中页的修改，首先修改缓冲池中的页，然后再以一定的频率刷新到磁盘(CheckPoint)；
innodb_buffer_pool_size；索引页、数据页、undo页、insert buffer、自适应哈希索引、锁信息、数据字典信息etc；innodb_buffer_pool_instances
![](http://7xvulr.com1.z0.glb.clouddn.com/20171130230709.png?attname=)
- LRU List、Free List和Flush List
//缓冲池管理算法；Latest Recent Used(上限时触发，类似Redis Ltrim/Rtrim，区别是新读取的页不直接放入LRU的首部，而是放入midpoint，大概3/8位置，new/old之分，innodb_old_blocks_time/innodb_old_blocks_pct)
数据库实例初启动时，LRU为空，则存放在Free List，当需要从缓冲池中分页时，首先从Free List查找可用空闲页，若有则将该页从Free List中删除，放至LRU List中，
否则淘汰LRU List尾页数据，分配给新的页。page made yound->LRU List的Old部分加入New部分。
Flush List，即脏页列表，脏页(dirty page)，LRU List中的页被修改后，即缓冲池中的页和磁盘中的页的数据产生了不一致。这时，数据库会通过CheckPoint机制将脏页刷新会磁盘。
```
select pool_id,hit_rate,pages_made_young,pages_not_made_young from information_schema.innodb_buffer_pool_stats;
select table_name,space,page_number,page_type from information_schema.innodb_buffer_page_lru where space =1;
```

- 重做日志缓冲redo log buffer 
//事务日志通过重做（redo）日志文件和innodb存储引擎的日志缓冲（innodb log buffer）来实现；
当开始一个事务时，会记录该事务的一个LSN(Log sequence number)，当事务执行时，会往innodb存储引擎的缓冲池里插入事务日志，
当事务提交时，将日志缓冲写入磁盘（默认的实现，即innodb_flush_log_at_trx_commit=1）,
也就是写数据前，需要先写日志，这种方式为预写日志方式（write-ahead logging,WAL），Master Thread每一秒将重做日志缓冲刷新到重做日志文件，
每个事务提交时会将重做日志缓冲刷新到重做日志文件，当重做日志缓冲池剩余空间小于1/2时，亦会触发重做日志刷新至重做日志文件。

- 额外的内存池
//缓冲池中的帧缓冲(frame buffer)、缓冲控制对象(buffer control block)记录了一些诸如LRU、锁、等待等信息，需要从额外内存池中申请

![](http://7xvulr.com1.z0.glb.clouddn.com/806053-20170831131657296-1607923143.png)

### Checkpoint

> 事务-Write Ahead Log策略，当事务提交时，先写重做日志(循环使用)，再修改页。当由于发生宕机而导致数据丢失时，通过重做日志来完成数据恢复(事务的Durability持久性要求)。当数据库运行了几个月甚至几年时，这时发生宕机，重新应用重做日志的时间会非常久，此时恢复的代价也会非常大。Checkpoint目的是解决以下几个问题：
- 缩短数据库的恢复时间
- 缓冲池不够用时，将脏页刷新到磁盘
- 重做日志不可用时，刷新脏页
> 当数据库发生宕机时，数据库不需要重做所有的日志，因为Checkpoint之前的页都已经刷新回磁盘，数据库只需对Checkpoint后的重做日志进行恢复，这就可以缩短恢复的时间。此外，当缓冲池不够用时，根据LRU算法会溢出最新最少使用的页，若此页为脏页，则强制执行Checkpoint，将脏页刷回磁盘。


