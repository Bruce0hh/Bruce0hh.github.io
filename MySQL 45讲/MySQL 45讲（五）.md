# 数据库表的空间回收

## 参数`innodb_file_per_table`

- **表数据既可以存在共享表空间里，也可以是单独的文件。**这个行为是由参数`innodb_file_per_table`控制：
  - **OFF**：表的数据放在系统共享表空间，也就是跟数据字典放在一起；
  - **ON**：每个InnoDB表数据存储在一个以`.ibd`为后缀的文件中。
- MySQL 5.6.6版本开始，默认为ON值。因为一个表单独存储为一个文件更容易管理，而且在你不需要这个表的时候，通过`drop table`命令，系统就会直接删除这个文件。而如果是放在共享表空间中，即使表删掉了，空间也不会回收的。
- 所以**推荐将`innodb_file_per_table`设置为ON。**
- 在删除整个表的时候，可以使用`drop table`命令回收表空间。但是，我们遇到更多的删除数据的场景是删除行，这时就遇到问题：表中的数据被删除了，但是表空间却没被回收。

## 数据删除流程

<img src="http://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/MySQL%E5%AE%9E%E6%88%9845%E8%AE%B2/assets/f0b1e4ac610bcb5c5922d0b18563f3c8.png" alt="img" style="zoom:50%;" />

- **数据页的复用和记录的复用是不同的**。
- **记录的复用，只限于符合范围条件的数据。**当R4这条记录被删除后，如果插入一个ID是400的行，可以直接复用这个空间。但如果插入ID是800的行，就不能复用这个位置。
- **数据页的复用**，就是当整个页从B+树里面摘掉了，可以复用到任何位置。假设pageA上所有记录被删除了，此时重新插入一条记录需要使用到新页，pageA是可以被复用的。如果相邻的两个数据页利用率都很小，系统就会把这两个页上的数据合到其中一个页上，另外一个数据页就被标记为可复用。
- 同样地，如果使用delete命令把整个表的数据删除，结果就是所有的数据页都会被标记为可复用。但是磁盘上，文件不会变小。所以**delete命令只是把记录的位置，或者数据页标记为可复用，但磁盘文件的大小是不会变的。即通过delete命令是不能回收表空间的。**

<img src="http://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/MySQL%E5%AE%9E%E6%88%9845%E8%AE%B2/assets/8083f05a4a4c0372833a6e01d5a8e6ea.png" alt="img" style="zoom:50%;" />

- **插入数据也会导致页分裂。**如果数据是按照索引递增顺序插入的，那么索引是紧凑的。但如果是随机插入的，可能会造成索引的数据页分裂。假设pageA满了，插入一个ID是550的数据，就不得不申请一个新的pageB来保存数据。当页分裂完成后，那么此时pageA的末尾会有空数据占用空间（注意：实际上，可能不止 1 个记录的位置是空洞）。
- 更新索引上的值，可以理解为删除一个旧的值，再插入一个新值。不难理解，这也是会造成空洞的。也就是说，经过大量增删改的表，都是可能是存在空洞的。所以，如果能够把这些空洞去掉，就能达到收缩表空间的目的。

## 重建表

- **重建表**：假如，对表A做空间收缩，可以新建一个与表A结构相同的表B，然后按照主键ID递增的顺序，把数据一行一行地从表A里读出来再插入到表B。可以使用`alter table A engine = InnoDB`命令来重建表。
- **MySQL 5.5版本之前**，上述重建表的命令执行流程和前面描述的差不多，区别是这个临时表B不需要你自己创建，MySQL会自动完成转存数据、交换表名、删除旧表的操作。显然，**在往临时表中插入数据的过程中，如果有数据插入到表A中的话，就会造成数据丢失**。因此整个DDL过程中，表A中不能有更新。
- 在**MySQL 5.6 版本开始引入的 Online DDL，对这个操作流程做了优化。**Online DDL步骤：
  1. 建立一个临时文件，扫描表A主键的所有数据页；
  2. 用数据页中表A的记录生成B+树，存储到临时文件中；
  3. 生成临时文件的过程中，将所有对A的操作记录在一个日志文件（`row log`）中；
  4. 临时文件生成后，将日志文件中的操作应用到临时文件，得到一个逻辑数据上与表A相同的数据文件；
  5. 用临时文件替换表A的数据文件。
- **Online DDL由于日志文件记录和重放操作这个功能的存在，这个方案在重建表的过程中，允许对表A做增删改的操作。**
- **关于锁：**alter语句在启动时候需要获取MDL写锁，但是这个写锁在真正拷贝数据之前就退化成读锁了。为了实现Online，MDL读锁不会阻塞增删改操作。不直接解锁是为了禁**止别的线程对这个表同时做DDL**。Online DDL最耗时的就是拷贝数据到临时表的过程，这其中可以接受增删改的操作。所以，相对于整个DDL过程来说，**锁的时间非常短**。对业务来说，就可以认为是Online的。
- 无论是否是Online，上述的重建方法都会扫描原表数据和构建临时文件。对于很大的表来说，这个操作是很消耗IO和CPU资源的。因此，如果是线上服务，你要很小心地控制操作时间，如果想要比较安全的操作的话，推荐使用GitHub开源的`gh-ost`来做。

## Online 和 inplace

- **改锁表DDL：**根据表A中的数据导出来的存放位置叫作`tmp_table`。这是一个临时表，在server层创建的。
- **Online DDL：**根据表A重建出来的数据是放在`tmp_file`里的，这个临时文件是InnoDB在内部创建出来的。**对于server层，没有把数据挪动到临时表，是一个原地操作，即inplace。**
- `tmp_file`是要占用临时空间的。`alter table t engine = InnoDB`，默认是使用`ALGORITHM=inplace`，当你使用`ALGORITHM=copy`的时候，就是表示强制拷贝表，对应的是改锁表DDL的流程。
- **但inplace和Online不是一个意思。**比如，给InnoDB表的一个字段加全文索引，写法就是：

```mysql
alter table t add FULLTEXT(field_name);
```

这个过程是inplace的，但会阻塞增删改操作，是非Online的。

- **Online和inplace关系**
  - DDL过程如果是Online的，就一定是inplace的。
  - inplace的DDL，不一定是Online的。添加全文索引（FULLTEXT index）和空间索引（SPATIAL index）就属于这种情况。

### 三种重建表方式

- MySQL 5.6开始，`alter table t engine = InnoDB`（recreate），默认Online DDL方式。
- `analyze table t`不是重建表，只是对表的索引信息做重新统计，没有修改数据，这个过程加了MDL读锁。
- `optimize table t`等于`recreate+analyze`。

### 特殊机制

- 在重建表的时候，InnoDB 不会把整张表占满，每个页留了 1/16 给后续的更新用。也就是说，其实重建表之后不是“最”紧凑的。

> 什么时候使用 alter table t engine=InnoDB 会让一个表占用的空间反而变大。

1. 将表T重建一次；
2. 插入一部分数据，但是插入的数据，用掉了一部分的预留空间；
3. 这种情况下，再次重建表T，就可能会出现问题中的现象

# count()

## count(*)的实现方式

- MyISAM引擎把表的总行数存在磁盘上，因此执行count(*)的时候，会直接返回这个数，效率很高。如果有过滤条件，MyISAM也不能返回那么快的；
- InnoDB引擎在执行count(*)的时候，需要把数据一行一行地从引擎里面读出来，然后累计计数。

### InnoDB的实现方式

- 由于InnoDB的事务设计，可重复读是它默认的隔离级别，在代码上就是通过多版本并发（MVCC）来控制实现的。每一行记录都要判断自己是否对这个会话可见，因此对于count(*)请求来说，InnoDB只好把数据一行一行地读出依次判断，可见的行才能够用于计算总行数。
- InnoDB是索引组织表，主键索引树的叶子节点是数据，而普通索引树的叶子节点是主键值。所以，普通索引树比主键索引树小很多。对于count(*)这样的操作，遍历哪个索引树得到的结果逻辑上是一样的。因此，优化器会找到最小的那棵树来遍历。**在保证逻辑正确的前提下，尽量减少扫描的数据量，是数据库系统设计的通用法则之一。**

### show table status

- `show table status`这个命令的输出结果里面也有一个`TABLE_ROWS`用于显示这个表当前有多少行。
- 实际上，`TABLE_ROWS`就是从这个采样估算得来的，因此它很不准，官方文档说误差可能达到40%~50%。

综上所述：

- MyISAM表虽然count(*)很快，但是不支持事务；
- `show table status`命令虽然很快，但是不准确；
- InnoDB表直接count(*)会遍历全表，虽然结果准确，但是会导致性能问题。

## 用缓存系统保存计数

- 使用Redis，但是缓存系统可能丢失更新。需要对数据化进行持久化存储，当Redis异常重启，虽然会丢失数据，重启后在数据库重新计算就好。
- 但实际上，**将计数保存在缓存系统中的方式，还不只是丢失更新的问题。即使 Redis 正常工作，这个值还是逻辑上不精确的。**
- **由于Redis和MySQL是两个不同的存储构成的系统，不支持分布式事务，无法拿到精确一致的视图。**并发系统里面，我们是无法精确控制不同线程的执行时刻的，所以多事务的时候无法统一视图。

## 在数据库保存计数

将上述用缓存系统，如Redis保存计数的方案换成用InnoDB表保存即可。利用InnoDB支持事务的特性，解决统一视图的问题。

## 不同的count用法

### count()语义

- **count()：**count()是一个聚合函数，对于返回的结果集，一行行地判断，如果count函数的参数不是NULL，累计值就加1，否则就不加，最后返回累计值。
- count(*)、count(主键id)、count(1)都表示返回满足条件的结果集的总行数；
- count(字段)，则表示返回满足条件的数据行里，参数“字段”不为NULL的总个数。

### 性能差别分析

1. server层要什么给什么；
2. InnoDB只给必要的值；
3. 现在的优化器只优化了count(*)的语义为“取行数”,其他优化并没有做。

### count()分析

- **count(主键id)：**InnoDB引擎会遍历整张表，把每一行的id值取出来，返回给server层。server层拿到id后，判断是不可能为空的，就按行累加。
- **count(1)：**InnoDB引擎遍历整张表，但不取值。server层对于返回的每一行，放一个数字“1”进去，判断是不可能为空的，按行累加。count(1)执行比count(主键id)快，因为从引擎返回id会涉及到解析数据行，以及拷贝字段值的操作。
- **count(字段)：**server层要什么字段，InnoDB就返回什么字段。
  1. 如果字段定义为not null的话，一行行地从记录里面读出这个字段，判断不能为null，按行累加；
  2. 如果允许为null的话，那么执行的时候，判断到有可能是null，还要把值取出来再判断一下，不是null才累加。
- **count(*)：**不会把全部字段取出来，而是专门做了优化，不取值。count(\*)肯定不是null，按行累加。
- **结论：**按照效率排序的话，count(字段)<count(主键 id)<count(1)≈count(*)。

# 日志和索引

## 日志

<img src="http://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/MySQL%E5%AE%9E%E6%88%9845%E8%AE%B2/assets/ee9af616e05e4b853eba27048351f62a.jpg" alt="img" style="zoom:50%;" />

### MySQL异常重启

- **在时刻A处发生crash：**在redo log处于prepare阶段之后，写binlog之前，发生了crash（崩溃）。由于此时binlog还没写，redo log也未提交，所以崩溃恢复的时候，这个事务会回滚。此时binlog还没写，所以备库也没有数据。
- **在时刻B处发生crash：**在binlog写完，但是redo log还没commit的时候发生crash。  这样，在崩溃恢复的时候，事务会被提交。
- **崩溃恢复的判断规则：**
  1. 如果redo log里面的事务是完整的，也就是有了commit的标识，则直接提交；
  2. 如果redo log里面的事务只有完整的prepare，则判断相应的binlog是否存在并完整：
     - binlog完整，提交事务。
     - 否则，回滚事务。

### MySQL如何知道binlog是完整的？

一个事务的binlog是有完整格式的：

- `statement`格式的binlog，最后会有COMMIT；
- `row`格式的binlog，最后会有一个XID event。
- 在MySQL 5.6.2版本以后，引入了`binlog-checksum`参数，用来验证binlog的正确性。

### redo log 和 binlog 如何关联？

- 两者有共同的数据字段，叫XID。
- 崩溃恢复的时候，会按顺序扫描redo log：
  - 如果既有prepare，又有commit的redo log，就直接提交。
  - 如果只有prepare，没有commit的redo log，就用XID去binlog找对应的事务。

### MySQL 的崩溃恢复设计？

通过处于prepare阶段的redo log 和完整的 binlog，重启就能恢复。这样的设计保证了数据备份的一致性。主库提交事务，binlog已经写入了，这个binlog也会被从库使用。

### 为什么需要两阶段提交？

对于InnoDB引擎来说，如果redo log提交完成了，事务就不能回滚，因为可能会覆盖掉别的事务更新。 而如果redo log直接提交，然后binlog写入的时候失败，InnoDB又回滚不了，数据和binlog日志又不一样了。

### 只使用binlog为何不可？

- 历史原因：**InnoDB并不是MySQL的原生存储引擎。**MyISAM设计之初并没有支持崩溃恢复。而InnoDB引擎作为MySQL插件加入MySQL引擎家族之前，就已经提供了一个崩溃恢复和事务支持的引擎了。
- 现实原因：**binlog没有能力恢复“数据页”。**在没有redo log，只有binlog的情况下，如果有两个事务A、B先后提交，在事务B，即第二个commit之前MySQL crash了，在使用binlog崩溃恢复的时候，事务A由于已经被认为完成了，binlog里面也没有记录数据页的更新细节，事务A是丢失了的。**如果使用binlog记录数据页的更新，其实就是又做了一个redo log出来。**

### 只使用redo log为何不可？

1. binlog有归档的作用。redo log是循环写，写到末尾是要回到开头继续写的。这样历史日志没法保留，redo log就起不到归档的作用。
2. MySQL依赖于binlog。binlog作为MySQL一开始就有的功能，被用在了许多地方，其中，MySQL系统高可用的基础，就是binlog复制。

### redo log 一般设置多大？

redo log太小的话，会导致很快就被写满，然后不得不强行刷redo log，那么WAL机制的能力就发挥不出来了。所以，如果是现在常见的几个TB的磁盘，可以直接将redo log设置为4个文件，每个文件1GB吧。

### 正常运行的实例，数据写入后的最终落盘，是从redo log更新的，还是从buffer pool更新过来的？

实际上，redo log并没有记录数据页完整数据，所以它并没有能力更新磁盘数据页。

1. 如果是正常运行的实例，数据页被修改后，跟磁盘的数据页不一致，称为脏页。**最终数据落盘，就是把内存中的数据页写盘，这个过程，与redo log毫无关系。**
2. 在崩溃恢复场景中，InnoDB如果判断到一个数据页可能在崩溃恢复的时候丢失了更新，就会将它堵到内存，然后让redo log更新内存内容。更新完成后，内存页变成脏页，就回到第一种状态。

### redo log buffer是什么？是先修改内存，还是先写redo log？

事务在做多个更新的时候，这个过程生成的日志都得先保存起来，但又不能在commit之前直接写到redo log文件里。所以使用redo log buffer，就是一块内存，用来先存redo日志的。但是真正把日志写到redo log文件（文件名是`ib_logfile`+数字），是在执行commit语句的时候做的。

## 更新数据的方式（验证）

```mysql
-- 创建表
mysql> CREATE TABLE `t` (
`id` int(11) NOT NULL primary key auto_increment,
`a` int(11) DEFAULT NULL
) ENGINE=InnoDB;
insert into t values(1,2);
-- 执行
mysql> update t set a=2 where id=1;
```

> MySQL内部是如何处理这个命令的？
>
> 1. 更新都是先读后写的，MySQL 读出数据，发现 a 的值本来就是 2，不更新，直接返回，执行结束；
> 2. MySQL 调用了 InnoDB 引擎提供的“修改为 (1,2)”这个接口，但是引擎发现值与原来相同，不更新，直接返回；
> 3. InnoDB 认真执行了“把这个值修改成 (1,2)"这个操作，该加锁的加锁，该更新的更新。

- **更新都是先读后写的，MySQL 读出数据，发现 a 的值本来就是 2，不更新，直接返回，执行结束；**

![img](http://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/MySQL%E5%AE%9E%E6%88%9845%E8%AE%B2/assets/6d9d8837560d01b57d252c470157ea90.png)

session2被blocked了，说明session有更新操作，而只有InnoDB才有加锁这个动作，排除选项1。

- **MySQL 调用了 InnoDB 引擎提供的“修改为 (1,2)”这个接口，但是引擎发现值与原来相同，不更新，直接返回；**

![img](http://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/MySQL%E5%AE%9E%E6%88%9845%E8%AE%B2/assets/441682b64a3f5dd50f35b12ca4b87c96.png)

- sessionA的第二个select语句返回了(1,3)，实际上这个语句是快照读，不能看见sessionB的更新。如果sessionA只是判断相同不更新，那么应该看到的是(1,2)。说明此时这个select语句，看到的是sessionA自己的update语句。排除选项2。

- 实际上，更新的逻辑是选项3。**InnoDB 认真执行了“把这个值修改成 (1,2)"这个操作，该加锁的加锁，该更新的更新。**

![img](http://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/MySQL%E5%AE%9E%E6%88%9845%E8%AE%B2/assets/63dd6df32dacdb827d256e5acb9837c1.png)

- MySQL根据条件字段判断记录是否需要修改，如果只读了部分信息，就没法确定是否需要修改；如果读出所有字段信息，就不会执行update。
- 验证结果都是在 binlog_format=statement 格式下进行的。 如果是 binlog_format=row 并且 binlog_row_image=FULL 的时候，由于 MySQL 需要在 binlog 里面记录所有的字段，所以在读数据的时候就会把所有数据都读出来了。
- 如果表中有 timestamp 字段而且设置了自动更新的话，那么更新“别的字段”的时候，MySQL 会读入所有涉及的字段，这样通过判断，就会发现不需要修改。
