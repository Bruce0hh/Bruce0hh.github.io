[TOC]

# MySQL 索引优化

## 优化器的逻辑

- **选择索引是优化器的工作。**而优化器选择索引的目的，是找到一个最优的执行方案，并用最小的代价去执行语句。
- 在数据库里面，扫描行数是影响执行代价的因素之一。扫描的行数越少，意味着访问磁盘数据的次数越少，消耗的CPU资源越少。
- 扫描行数并不是唯一的判断标准，优化器还会结合是否使用临时表、是否排序等因素进行综合判断。

### 扫描行数的判断

- MySQL在真正开始执行语句之前，并不能精确地满足这个条件的记录有多少条，而只能根据统计信息来估算记录数。这个统计信息就是**区分度**。
- 显然，一个索引上不同的值越多，这个索引的区分度就越好。而一个索引上不同的值的个数，我们称之为**基数**（cardinality）。也就是说，这个基数越大，索引的区分度越好。

### MySQL 索引基数

- 因为把整张表取出来一行行统计，虽然可以得到精确的结果，但是代价太高了，所以只能选择“采样统计”。
- 采样统计的时候，InnoDB默认会选择N个数据页，统计这些页面上的不同值，得到一个平均值，然后乘以这个索引的页面数，就得到了这个索引的基数。
- 而数据表是会持续更新的，索引统计信息也不会固定不变。所以，当变更的数据行数超过1/M的时候，会自动触发重新做一次索引统计。
- 在MySQL中，有两种存储索引统计的方式，可以通过设置参数`innodb_stats_persisitent`的值来选择：
  1. 设置为on的时候，表示统计信息会持久化存储。这时，默认的N是20，M是10。
  2. 设置为off的时候，表示统计信息只存储在内存中。这时，默认的N是8，M是16。
- 由于是采样统计，所以不管是20还是8，这个基数都是很容易不准的。其实索引统计只是一个输入，对于一个具体的语句来说，优化器还要判断，执行这个语句本身要扫描多少行。

- 如果explain的结果预估的rows值跟实际情况差距比较大，可以通过使用`analyze table t`命令，重新统计索引信息。

## 索引选择异常和处理

```mysql
mysql> explain 
select * from t 
where (a between 1 and 1000) 
and 
(b between 50000 and 100000) 
order by b 
limit 1;
```

- 上述例子中，MySQL选择了先b索引，再a索引。导致扫描的行数从1000升到50000以上。

三种方法处理索引选择：

1. 使用`force index`语句，强制使用特定索引。
2. 修改语句，引导MySQL使用我们期望的索引。如上述例子，将`order by b limit 1`改成`order by b,a limit 1`，语义的逻辑是相同的。
3. 新建一个更合适的索引，来提供给优化器做选择，或者删掉误用的索引。

# 字符串字段索引

## 前缀索引

```mysql
-- 创建表
mysql> create table SUser(
ID bigint unsigned primary key,
email varchar(64), 
... 
)engine=innodb;

-- 创建索引
mysql> alter table SUser add index index1(email);
或
mysql> alter table SUser add index index2(email(6));
```

- 第一个语句创建的index1索引里面，包含了每个记录的整个字符串；而第二个语句创建的index2索引里面，对于每个记录都是只取前6个字节。
- MySQL是支持前缀索引的，也就是说，你可以定义字符串的一部分作为索引。默认地，如果你创建索引的语句不指定前缀长度，那么索引就会包含整个字符串。
- 由于index2这个索引结构都只取前6个字节，所以占用空间更小，这就是前缀索引的优势。同时，使用前缀索引后，可能由于区分度不高，会导致查询语句读数据的次数变多。
- 在建立索引时关注的是区分度，区分度越高越好。因为区分度越高，意味着重复的键值越少。因此，我们可以通过统计索引上有多少个不同的值来判断要使用多长的前缀。
- **使用前缀索引，定义好长度，就可以做到既节省空间，又不用额外增加太多的查询成本。**

```mysql
-- 计算这个列上有多少个不同的值
mysql> select count(distinct email) as L from SUser;

-- 依次选取不同的长度的前缀来看这个值，比如我们要看一下4~7个字节的前缀索引
mysql> select 
  count(distinct left(email,4)）as L4,
  count(distinct left(email,5)）as L5,
  count(distinct left(email,6)）as L6,
  count(distinct left(email,7)）as L7,
from SUser;
```

- 预先设定一个可以接受的损失的比例，比如5%。然后在返回的L4~L7之中，找出不小于`L*95%`的值，假设这里L6和L7都满足，就选择前缀长度为6。

## 前缀索引对覆盖索引的影响

```mysql
-- 只返回id,email字段
select id,name,email from SUser where email='zhangssxyz@xxx.com';
```

- 如果只使用index1的话，可以使用覆盖索引，从index1查到结果后就直接返回了，不需要回到ID索引再查一次。
- 如果只用index2索引的话，就不得不回到ID索引去判断email字段的值，即使你将index2定义为email(18)的前缀索引，虽然此时index2已经包含所有信息，但InnoDB还是要回到ID索引查一下，因为系统并不确定前缀索引的定义是否截断了完整信息。
- 所以，**使用前缀索引就用不上覆盖索引对查询性能的优化了**。

## 其他方式

对于类似身份证这种字段，前缀的区分度不高，我们可采用以下两种方法：

1. **倒序存储**，由于身份证号的最后6位没有地址码这样的重复逻辑，所以最后这6位就很可能提供了足够的区分度。

```mysql
-- 类似的，你可以倒序存储
mysql> select field_list from t where id_card = reverse('input_id_card_string');
```

2. **hash字段**，你可以在表上再创建一个整数字段，来保存身份证的校验码，同时在这个字段上创建索引。然后，每次插入新纪录的时候，都同时用crc32()这个函数得到校验码填到这个新字段。由于校验码可能存在冲突，也就是说两个不同的身份证号通过crc32()函数得到的结果可能是相同的，所以你的查询语句where部分要判断id_card的值是否精确相同。

```mysql
-- 创建校验码字段
mysql> alter table t add id_card_crc int unsigned, add index(id_card_crc);

-- 判断id_card是否精确相同
mysql> select field_list from t 
where id_card_crc=crc32('input_id_card_string') 
and id_card='input_id_card_string'
```

**倒序存储和使用hash字段异同点**：

1. **占用额外空间**：**倒序存储**方式在主键索引上，不会消耗额外的存储空间，而**hash字段方法**需要增加一个字段。当然，倒序存储方式使用4个字节的前缀长度应该是不够的，如果再长一点，这个消耗跟额外的hash字段也差不多抵消了。
2. **CPU消耗**：**倒序方式**每次读写的时候，都需要额外调用一次reverse函数，而**hash字段**的方式需要额外调用一次crc32()函数。如果只从这两个函数的计算复杂度来看的话，reverse 函数额外消耗的CPU资源会更小一些。
3. **查询效率**：**倒序存储**方式用的是前缀索引的方式，就是还会增加扫描行数。使用**hash字段**的查询性能相对更稳定一些。因为算出来的值虽然有冲突的概率，但是概率非常小，可以认为每次查询的平均扫描行数接近1。

## 总结

1. 直接创建完整索引，这样可能比较占用空间；
2. 创建前缀索引，节省空间，但会增加查询扫描次数，并且不能使用覆盖索引；
3. 倒序存储，再创建前缀索引，用于绕过字符串本身前缀的区分度不够的问题；
4. 创建 hash 字段索引，查询性能稳定，有额外的存储和计算消耗，跟第三种方式一样，都不支持范围扫描。

# MySQL的WAL机制

## MySQL更新语句

- InnoDB在处理更新语句的时候，只做了写日志这一个磁盘操作。这个日志叫做 `redo log`（重做日志），在更新内存写完`redo log`后，就返回给客户端，本次更新成功。
- **当内存数据页跟磁盘数据页内容不一致的时候，我们称这个内存页为“脏页”。内存数据写入到磁盘后，内存和磁盘上的数据页的内容就一致了，称为“干净页”**。
- 平时执行很快的更新操作，其实就是在写内存和日志，如果突然执行很慢，可能就是在刷脏页（flush）。

### 刷脏页场景：

1. **InnoDB的`redo log`写满了。**这时候系统会停止所有更新操作，把`checkpoint`往前推进，`redo log`留出空间可以继续写。` checkpoint`可不是随便往前修改一下位置就可以的。移动的两点之间的距离，就是日志，所对应的脏页全都flush到磁盘上。之后，从`write pos`的位置到`checkpoint`的位置之间就是可以再写入的`redo log`的区域。
2. **系统内存不足。**当需要新的内存页，而内存不够用的时候，就要淘汰一些数据页，空出内存给别的数据页使用。如果淘汰的是”脏页“，就要将脏页写到磁盘。如果刷脏页一定会写盘，就保证了每个数据页有两种状态：
   - 内存里存在，内存里就肯定是正确的结果，直接返回；
   - 内存里没有数据，就可以肯定数据文件上是正确的结果，读入内存后返回。这样效率更高。
3. **MySQL认为系统“空闲”的时候。**系统没什么压力，MySQL空闲时的操作。
4. **MySQL正常关闭的时候。**这时候，MySQL会把内存的脏页都flush到磁盘上，这样下次MySQL启动的时候，就可以直接从磁盘上读数据，启动速度会很快。

### 四种场景刷脏页性能：

1. 第三种场景和第四种场景没什么性能问题。

2. InnoDB要尽量避免第一种场景。因为此时整个系统就不能再接受更新了，所有的更新都必须堵住。从监控上看，这时更新数会跌为0。

3. 第二种场景是常态。InnoDB用缓冲池（buffer pool）管理内存，缓冲池中的内存页有三种状态：

   - 还没有使用
   - 使用了并且是干净页
   - 使用了并且是脏页

4. InnoDB的策略是尽量使用内存，因此对于一个长时间运行的库来说，未被使用的页面很少。而当要读入的数据页没有在内存的时候，就必须到缓冲池中申请一个数据页。这时候只能把最久不使用的数据页从内存中淘汰掉：如果淘汰的是一个干净页，就直接释放出来复用；但如果是脏页，就必须将脏页先刷到磁盘，变成干净页后才能复用。

5. 虽然刷脏页是常态，出现以下两种情况，都是会明显影响性能的：

   - 一个查询要淘汰的脏页个数太多，会导致查询的响应时间明显变长；
   - 日志写满，更新全部堵住，写性能跌为0，对于敏感业务来说，是不能接受的。

   所以，InnoDB需要有控制脏页比例的机制，来尽量避免上面的情况。

## InnoDB刷脏页的控制策略

### 刷脏页参数&性能

- 使用`innodb_io_capacity`参数，它会告诉你InnoDB磁盘能力。建议设置为磁盘的IOPS。磁盘的IOPS可以通过fio这个工具来测试，下面的语句是用来测试磁盘随机读写的命令：

```mysql
fio -filename=$filename -direct=1 -iodepth 1 -thread -rw=randrw -ioengine=psync -bs=16k -size=500M -numjobs=10 -runtime=10 -group_reporting -name=mytest 
```

- 没能正确地设置`innodb_io_capacity`参数，也经常可能导致性能问题。比如MySQL的写入速度很慢，TPS很低，但是数据库主机的IO压力并不大，就有可能是这个参数设置有问题。
- 如果你的磁盘是SSD，但是`innodb_io_capacity`设置的值为300。于是，InnoDB认为这个系统的能力就这么差，所以刷脏页特别慢，甚至比脏页生成的速度还慢，这样就造成了脏页的累积，影响了查询和更新性能。

### 刷脏页策略

- InnoDB的刷盘速度参考两个因素：一个是**脏页比例**，一个是**redo log写盘速度**。InnoDB会根据这两个因素先单独算出两个数字。
- 参数`innodb_max_dirty_pages_pct`是脏页比例上限，默认值75%。InnoDB会根据当前的脏页比例（假设为M），算出一个范围在0到100之间的数字，计算这个数字的伪代码类似这样：

```mysql
F1(M)
{
  if M>=innodb_max_dirty_pages_pct then
      return 100;
  return 100*M/innodb_max_dirty_pages_pct;
}
```

- InnoDB每次写入的日志都有一个序号，当前写入的序号跟`checkpoint`对应的序号之间的差值，假设为N。InnoDB会根据这个N算出一个范围在0到100之间的数字，这个计算公式可以记为F2(N)。N越大，算出来的值越大。然后根据上述算得的F1(M)和F2(N)两个值，取其中较大的值记为R（`R=max{F1, F2}`），之后引擎就可以按照`innodb_io_capacity`定义的能力乘以R%来控制刷脏页的速度。
- InnoDB会在后台刷脏页，而刷脏页的过程是要将内存页写入磁盘。所以，无论是你的**查询语句需要内存的时候可能要求淘汰一个脏页**，还是由于**刷脏页的逻辑会占用IO资源并可能影响到了你的更新语句**，都可能是造成你从业务端感知到执行语句变慢。
- 要尽量避免这种情况，你就要合理地设置`innodb_io_capacity`的值，并且平时要多关注脏页比例，不要让它经常接近75%。脏页比例是通过`Innodb_buffer_pool_pages_dirty / Innodb_buffer_pages_total`得到的：

```mysql
mysql> 
select VARIABLE_VALUE into @a from global_status where VARIABLE_NAME = 'Innodb_buffer_pool_pages_dirty';

select VARIABLE_VALUE into @b from global_status where VARIABLE_NAME = 'Innodb_buffer_pool_pages_total';

select @a/@b;
```

- 一旦一个查询请求需要在执行过程中先flush掉一个脏页时，这个查询就可能要比平时慢了。而MySQL中的一个机制，可能会让你的查询更慢：在准备刷一个脏页的时候，如果这个数据页旁边的数据页刚好是脏页，就会把这个旁边的脏页也带着刷掉，这个逻辑能一直蔓延下去。
- 在InnoDB中，`innodb_flush_neighbors`参数就是用来控制这个行为的，值为1的时候会有上述的“连坐”机制，值为0时表示不找旁边的脏页，只刷自己的。
- 实际上这个机制在机械硬盘时代是很有意义的，可以减少很多随机IO。机械硬盘的随机IOPS一般只有几百，相同的逻辑操作减少随机IO，就意味着系统性能的大幅度提升。
- 如果使用的是SSD这类IOPS比较高的设备的话，我就建议你把`innodb_flush_neighbors`的值设置为0。因为这时候IOPS往往不是瓶颈，而只刷自己，就能更快地执行完必要的刷脏页操作，减少SQL语句响应时间。
- MySQL8.0中，`innodb_flush_neighbors`参数的默认值已经是0了。

## 问题

> 如果一个高配的机器，redo log设置得太小，会发生什么情况。

- 每次事务提交都要写`redo log`，如果设置得太小，很快就会被写满。`write pos`一直追着`checkpoint`。这时候系统不得不停止所有更新，去推进`checkpoint`。
- 此时看到的现象就是，**磁盘压力很小，但是数据库出现间歇性的性能下跌。**

