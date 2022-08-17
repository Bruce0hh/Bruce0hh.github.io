[TOC]

# MySQL 的日志系统

## binlog的写入机制

- **binlog的写入逻辑：**事务执行过程中，先把日志写到`binlog cache`，事务提交的时候，再把`binlog cache`写到binlog文件中。
- 一个事务的binlog是不能被拆开的，因此不论这个事务多大，也要确保一次性写入。这就涉及到了`binlog cache`所占内存的大小。如果超过了参数所规定的大小，就要暂存到磁盘。
- 事务提交的时候，执行器把`binlog cache`里的完整事务写入到binlog中，并清空`binlog cache`。
- 每个线程都有自己的`binlog cache`，但是共用同一份binlog文件。
- **binlog的write**，指的就是把日志写入到文件系统的`page cache`，并没有把数据持久化到磁盘，所以比较快。
- **binlog的fsync**，才是将数据持久化到磁盘的操作。一般情况下，我们认为fsync才占磁盘的IOPS。

write和fsync的时机，是由参数`sync_binlog`控制的：

1. `sync_binlog=0`的时候，表示每次提交事务都只write，不fsync。
2. `sync_binlog=1`的时候，表示每次提交事务都会执行fsync。
3. `sync_binlog=N(N>1)`的时候，表示每次提交事务都write，但累计N个事务后才fsync。

> 因此，在出现 IO 瓶颈的场景里，将 sync_binlog 设置成一个比较大的值，可以提升性能。在实际的业务场景中，考虑到丢失日志量的可控性，一般不建议将这个参数设成 0，比较常见的是将其设置为 100~1000 中的某个数值。
> 
> 但是，将 sync_binlog 设置为 N，对应的风险是：如果主机发生异常重启，会丢失最近 N 个事务的 binlog 日志。

## redo log的写入机制

**redo log的存储状态：**

1. 存在`redo log buffer`中，物理上是在MySQL进程内存中；
2. 写到磁盘write，但是没有持久化fsync，物理上是在文件系统的`page cache`里面；
3. 持久化到磁盘，对应的是`hard disk`。

日志写到`redo log buffer`是很快的，write到`page cache`也差不多，但是持久化到磁盘的速度就慢多了。

为了控制redo log的写入策略，InnoDB提供了`innodb_flush_log_at_trx_commit`参数，它有三种可能取值：

1. 设置为0的时候，表示每次事务提交时都只是把redo log留在`redo log buffer`中；
2. 设置为1的时候，表示每次事务提交时都将redo log直接持久化到磁盘；
3. 设置为2的时候，表示每次事务提交时都只是把redo log写到`page cache`。

> InnoDB 有一个**后台线程**，每隔 1 秒，就会把 redo log buffer 中的日志，调用 write 写到文件系统的 page cache，然后调用 fsync 持久化到磁盘。
> 
> 注意，事务执行中间过程的 redo log 也是直接写在 `redo log buffer` 中的，这些 redo log 也会被后台线程一起持久化到磁盘。也就是说，**一个没有提交的事务的 redo log，也是可能已经持久化到磁盘的。**

实际上，除了后台每秒一次的轮询操作外，还有**两种场景会让一个没有提交的事务的redo log写入到磁盘中**。

1. 一种是，**`redo log buffer`占用的空间即将达到`innodb_log_buffer_size`一半的时候**，后台线程会主动写盘。注意，由于这个事务并没有提交，所以这个写盘动作只是write，而没有调用fsync，也就是只留在了文件系统的`page cache`。
2. 另一种是，**并行的事务提交的时候，顺带将这个事务的`redo log buffer`持久化到磁盘**。假设一个事务A执行到一半，已经写了一些redo log到buffer中，这时候有另外一个线程的事务B提交，如果`innodb_flush_log_at_trx_commit`设置的是1，那么按照这个参数的逻辑，事务B要把`redo log buffer`里的日志全部持久化到磁盘。这时候，就会带上事务A在`redo log buffer`里的日志一起持久化到磁盘。

> 时序上redo log先prepare，再写binlog，最后再把redo log commit。
> 
> 如果把 `innodb_flush_log_at_trx_commit` 设置成 1，那么 redo log 在 prepare 阶段就要持久化一次，因为有一个崩溃恢复逻辑是要依赖于 prepare 的 redo log，再加上 binlog 来恢复的。
> 
> 每秒一次后台轮询刷盘，再加上崩溃恢复这个逻辑，InnoDB 就认为 redo log 在 commit 的时候就不需要 fsync 了，只会 write 到文件系统的 `page cache` 中就够了。

## 组提交

### LSN（log sequence number）日志逻辑序列号

- LSN是单调递增的，用来对应redo log的一个个写入点。每次写入长度为length的redo log，LSN的值就会加入length。
- LSN也会写到InnoDB的数据页中，来确保数据页不会被多次执行重复的redo log。

### redo log组提交

1. 多个并发事务，会选择第一个到达的成为leader；
2. 等leader开始写盘的时候，带的是最大LSN值MAX；
3. 在leader返回时，所有LSN小于等于MAX的事务redo log，都已经被持久化到磁盘；
4. 被持久化的redo log对应的事务可以直接返回了。

所以，一次组提交里面，组员越多，节约磁盘IOPS的效果越好。在并发更新场景下，第一个事务写完`redo log buffer`以后，接下来fsync越晚调用，组员可能越多，节约IOPS的效果越好。

<img src="http://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/MySQL%E5%AE%9E%E6%88%9845%E8%AE%B2/assets/5ae7d074c34bc5bd55c82781de670c28.png" alt="img" style="zoom:50%;" />

同样地，binlog也可以组提交。由于上述图中的第3步执行得很快，所以 binlog 的 write 和 fsync 间的间隔时间短，导致能集合到一起持久化的 binlog 比较少，因此 binlog 的组提交的效果通常不如 redo log 的效果那么好。

### binlog组提交

如果你想提升 binlog 组提交的效果，可以通过设置 `binlog_group_commit_sync_delay` 和 `binlog_group_commit_sync_no_delay_count` 来实现。

1. `binlog_group_commit_sync_delay` 参数，表示延迟多少微秒后才调用 fsync;
2. `binlog_group_commit_sync_no_delay_count` 参数，表示累积多少次以后才调用 fsync。

上述两个条件是或的关系，也就是说只有一个满足条件就会调用fsync。

所以，当`binlog_group_commit_sync_delay`设置为 0 的时候，`binlog_group_commit_sync_no_delay_count` 也无效了。

## 总结

**WAL机制优势：**

1. redo log 和 binlog 都是顺序写，磁盘的顺序写比随机写速度要快；
2. 组提交机制，可以大幅度降低磁盘的 IOPS 消耗。

**MySQL性能提升：**

1. 设置 `binlog_group_commit_sync_delay` 和 `binlog_group_commit_sync_no_delay_count` 参数，减少 binlog 的写盘次数。这个方法是基于“额外的故意等待”来实现的，因此可能会增加语句的响应时间，但没有丢失数据的风险。
2. 将 `sync_binlog` 设置为大于1的值（比较常见是 100~1000）。这样做的风险是，主机掉电时会丢binlog日志。
3. 将 `innodb_flush_log_at_trx_commit` 设置为 2。这样做的风险是，主机掉电的时候会丢数据。

**MySQL的crash-safe：**

1. 如果客户端收到事务成功的消息，事务就一定持久化了；
2. 如果客户端收到事务失败（比如主键冲突、回滚等）的消息，事务就一定失败了；
3. 如果客户端收到“执行异常”的消息，应用需要重连后通过查询当前状态来继续后续的逻辑。此时数据库只需要保证内部（数据和日志之间，主库和备库之间）一致就可以了。

**“非双1”场景：**

1. 业务高峰期。一般如果有预知的高峰期，DBA有预案，把主库设置成“非双1”。
2. 备库延迟，为了让备库尽快赶上主库。
3. 用备份恢复主库的副本，应用binlog的过程，同上一个场景类似。
4. 批量导入数据的时候。

一般情况，“非双1”配置：`innodb_flush_logs_at_trx_commit=2`，`sync_binlog=1000`。

# MySQL 主备原理

## 基本原理

<img src="http://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/MySQL%E5%AE%9E%E6%88%9845%E8%AE%B2/assets/fd75a2b37ae6ca709b7f16fe060c2c10.png" alt="img" style="zoom:50%;" />

状态1中，客户端的读写都直接访问节点A，而节点B是A的备库，只是将A的更新都同步过来，到本地执行。这样就可以保持节点B和A的数据是相同的。切换时，就切成状态2。在状态1中，虽然节点B没有被直接访问，但是仍然建议把节点B设置为只读模式。有以下几个考虑：

1. 有时候运营类的查询语句会被放到备库上去查，设置为只读可以防止误操作；
2. 防止切换逻辑有bug，比如切换过程中出现双写，造成主备不一致；
3. 可以用readonly状态，来判断节点的角色。

<img src="http://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/MySQL%E5%AE%9E%E6%88%9845%E8%AE%B2/assets/a66c154c1bc51e071dd2cc8c1d6ca6a3.png" alt="img" style="zoom:50%;" />

> readonly 设置对超级 (super) 权限用户是无效的，而用于同步更新的线程，就拥有超级权限。主库接收到客户端的更新请求后，执行内部事务的更新逻辑，同时写 binlog。备库 B 跟主库 A 之间维持了一个长连接。主库 A 内部有一个线程，专门用于服务备库 B 的这个长连接。

**流程：**

1. 在备库B上通过`change master`命令，设置主库A的IP、端口、用户名、密码，以及要从哪个位置开始请求binlog，这个位置包含文件名和日志偏移量。
2. 在备库B上执行`staret slave`命令，这时候备库会启动两个线程，就是图中的`io_thread`和`sql_thread`。其中`io_thread`负责与主库建立连接。
3. 主库A校验完用户名、密码后，开始按照备库B传过来的位置，从本地读取binlog，发给B。
4. 备库B拿到binlog后，写到本地文件，称为中转日志（relay log）。
5. `sql_thread`（后为多个线程）读取中转日志，解析出日志里的命令，并执行。

## binlog的三种格式（statement、row、mixed）

1. 当**binlog中设置的是statement格式，binlog记录的是SQL语句原文。delete带limit的语句**，可能会造成主备数据不一致的情况。因为**主备两次使用的索引可能不一样，导致使用limit的结果也不一致。**
2. 当使用了**row格式**，binlog没有了SQL语句的原文，而是替换成了两个event：**`Table_map`和`Delete_rows`**。binlog里面记录了真实删除行的主键id，这样当binlog传到备库去的时候，就会删除固定id的行，不会有主备删除不同行的里面。

### mixed格式

> 为什么会有mixed格式的binlog？

1. 因为statement格式的binlog可能会导致主备不一致，所以要用row格式。
2. 但是row格式很占空间，statement记录只是一条SQL语句，但是row要记录所有相关的数据行，同时写binlog也要耗费IO资源，影响执行速度。
3. 启用mixed格式时，MySQL自己会判断这条SQL语句是否可能会引起主备不一致，如果可能会不一致，就使用row格式，否则使用statement格式。

## 循环复制

<img src="http://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/MySQL%E5%AE%9E%E6%88%9845%E8%AE%B2/assets/20ad4e163115198dc6cf372d5116c956.png" alt="img" style="zoom:50%;" />

第一幅图为**M-S结构**，但在生产更多的是上图的**双M结构**。其实区别只是多了一条线，即：节点 A 和 B 之间总是互为主备关系。这样在切换的时候就不用再修改主备关系。

> 业务逻辑在节点 A 上更新了一条语句，然后再把生成的 binlog 发给节点 B，节点 B 执行完这条更新语句后也会生成 binlog。（我建议你把参数 log_slave_updates 设置为 on，表示备库执行 relay log 后生成 binlog）。
> 
> 那么，如果节点 A 同时是节点 B 的备库，相当于又把节点 B 新生成的 binlog 拿过来执行了一次，然后节点 A 和 B 间，会不断地循环执行这个更新语句，也就是循环复制了。

**规定（使用server id防止循环复制）：**

1. 规定两个库的 server id 必须不同，如果相同，则它们之间不能设定为主备关系；
2. 一个备库接到 binlog 并在重放的过程中，生成与原 binlog 的 server id 相同的新的 binlog；
3. 每个库在收到从自己的主库发过来的日志后，先判断 server id，如果跟自己的相同，表示这个日志是自己生成的，就直接丢弃这个日志。

**流程：**

1. 从节点 A 更新的事务，binlog 里面记的都是 A 的 server id；
2. 传到节点 B 执行一次以后，节点 B 生成的 binlog 的 server id 也是 A 的 server id；
3. 再传回给节点 A，A 判断到这个 server id 与自己的相同，就不会再处理这个日志。所以，死循环在这里就断掉了。

# MySQL高可用

## 主备延迟

### 同步延迟

1. 主库A执行完成一个事务，写入binlog，我们把这个时刻记为T1；
2. 之后传给备库B，我们把备库B接受完这个binlog的时刻记为T2；
3. 备库B执行完成这个事务，我们把这个时刻记为T3。

主备延迟，就是同一个事务，在备库执行完成的时间和主库执行完成的时间之间的差值，也就是T3-T1。通过在备库上执行`show slave status`命令，它的返回结果里面会显示`seconds_behind_master`，用于表示当前备库延迟了多少秒。

`seconds_behind_master`的计算方法：

1. 每个事务的binlog里面都有一个时间字段，用于记录主库上写入的时间；
2. 备库取出当前正在执行的事务的时间字段的值，计算它与当前系统时间的差值，得到`seconds_behind_master`。就是上述的T3-T1。

> 如果主备库机器的系统时间设置不一致，会不会导致主备延迟的值不准？

答案是不会的。因为当备库连接到主库的时候，会通过`select unix_timestamp()`函数来获得当前主库的系统时间。如果这时候发现主库的系统时间与自己不一致，备库在执行`seconds_behind_master`计算的时候会自动扣掉这个差值。

- 在网络正常的时候，日志从主库传给备库所需的时间是很短的，即T2-T1的值是非常小的。也就是说，在网络正常的情况下，主备延迟的主要来源是备库接收玩binlog和执行完这个事务之间的时间差。
- 所以，**主备延迟最直接的表现是，备库消费中转日志（relay log）的速度，比主库生产binlog的速度要慢。**

## 主备延迟的来源/原因

1. **备库所在机器的性能要比主库所在的机器性能差。**
   - 更新请求对于IOPS的压力，在主库和备库上是无差别的。所以，**这种部署一般会将备库设置为“非双1”的模式。**实际上，更新过程中也会触发大量的读操作。所以，当备库主机上的多个备库都在争抢资源的时候，就可能会导致主备延迟了。
   - 现在的部署通常选用相同规格的机器，并且做对称部署。
2. **备库压力大。**
   - 由于主库直接影响业务，使用比较克制，反而忽视了备库的压力控制。备库上的查询耗费了大量的CPU资源，影响了同步速度，造成了主备延迟。
   - **一主多从**可以让这些从库分担读的压力；通**过binlog输出到外部系统**，比如Hadoop，让外部系统提供统计类查询的能力。
   - 一主多从的方式大都会被采用。因为作为数据库系统，还必须保证有定期全量备份的能力。而从库，就很适合用来做备份。
3. **大事务**（如，一次性地用delete语句删除太多数据）。
   - 因为主库上必须等事务执行完成才会写入binlog，再传给备库。所以，如果一个主库上的语句执行10分钟，那这个事务很可能就会导致从库延迟10分钟。
4. **备库的并行复制能力。**

## 主备切换策略

### 可靠性优先策略

**双M结构的切换过程**：

1. 判断备库B现在的`seconds_behind_master`，如果小于某个值（比如5秒）就继续下一步，否则持续重试这一步；
2. 把主库A改成只读状态，即把`readonly`设置为true；
3. 判断备库B的`seconds_behind_master`的值，直到这个值变成0为止；
4. 把备库B改成可读写状态，也就是把`readonly`设置为false；
5. 把业务请求切到备库B。

> 在步骤2之后，主库A和备库B都处于`readonly`状态，也就是系统处于不可写状态，直到步骤5完成后才恢复。在不可用的状态中，比较费时的是步骤3，需要耗费好几秒的时间，这也是为什么需要步骤1做判断确保`seconds_behind_master`的值足够小。

### 可用性优先策略

> 将上述主备切换流程的步骤4、5调整到最开始执行，也就是说不等主备数据同步，直接把连接切到备库B，并且让备库B可以读写，那么系统几乎就没有不可用时间了。这个过程可能会导致数据不一致的情况。

**特点**

1. 使用row格式的binlog时，数据不一致的问题更容易被发现。而使用mixed或者statement格式的binlog时，数据很可能悄悄地就不一致了。
2. 主备切换的可用性优先策略会导致数据不一致。因此，大多数情况下都建议使用可靠性优先策略。

**可用性策略优先级更高的场景**

- 有一个库的作用是记录操作日志。此时，数据不一致可以通过binlog来修补，而这个短暂的不一致也不会引发业务问题。同时，业务系统依赖于这个日志写入逻辑，如果这个库不可写，会导致线上的业务操作无法执行。

### 总结

- 在满足数据可靠性的前提下，MySQL 高可用系统的可用性，是依赖于主备延迟的。延迟的时间越小，在主库故障的时候，服务恢复需要的时间就越短，可用性就越高。
