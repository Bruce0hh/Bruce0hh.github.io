[TOC]

# ORDER BY

```mysql
-- 创建表
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `city` varchar(16) NOT NULL,
  `name` varchar(16) NOT NULL,
  `age` int(11) NOT NULL,
  `addr` varchar(128) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `city` (`city`)
) ENGINE=InnoDB;

-- order by 语句
select city,name,age from t where city='杭州' order by name limit 1000;
```

## 全字段排序

- 使用explain命令查看语句的执行情况。发现Extra字段中包含`Using filesort`，就是需要排序，MySQL会给每个线程分配一块内存用于排序，称为`sort_buffer`。

### 语句执行流程

1. 初始化`sort_buffer`，确定放入name、city、age这三个字段；
2. 从索引city找到第一个满足`city=杭州`条件的主键id；
3. 到主键id索引取出整行，取name、city、age三个字段的值，存入`sort_buffer`中；
4. 从索引city取下一个记录的主键id；
5. 重复步骤3、4直到city的值不满足查询条件为止；
6. 对`sort_buffer`中的数据按照name做快速排序；
7. 按照排序结果取前1000行返回给客户端。

<img src="http://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/MySQL%E5%AE%9E%E6%88%9845%E8%AE%B2/assets/6c821828cddf46670f9d56e126e3e772.jpg" alt="img" style="zoom:50%;" />

- 图中的“按name”排序这个动作，可能是在内存中完成，也可能需要使用外部排序，这取决于排序所需的内存和参数`sort_buffer_size`。
- `sort_buffer_size`，也就是MySQL为排序开辟的内存（`sort_buffer`）的大小。如果要排序的数据量小于`sort_buffer_size`，排序就在内存中完成。但如果排序数据量太大，内存放不下，则不得不利用磁盘临时文件辅助排序。

通过以下的方法，确定一个排序语句是否使用了临时文件：

```mysql
/* 打开 optimizer_trace，只对本线程有效 */
SET optimizer_trace='enabled=on'; 
 
/* @a 保存 Innodb_rows_read 的初始值 */
select VARIABLE_VALUE into @a from  performance_schema.session_status where variable_name = 'Innodb_rows_read';
 
/* 执行语句 */
select city, name,age from t where city='杭州' order by name limit 1000; 
 
/* 查看 OPTIMIZER_TRACE 输出 */
SELECT * FROM `information_schema`.`OPTIMIZER_TRACE`\G
 
/* @b 保存 Innodb_rows_read 的当前值 */
select VARIABLE_VALUE into @b from performance_schema.session_status where variable_name = 'Innodb_rows_read';
 
/* 计算 Innodb_rows_read 差值 */
select @b-@a;
```

![全排序的 OPTIMIZER_TRACE 部分结果](http://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/MySQL%E5%AE%9E%E6%88%9845%E8%AE%B2/assets/89baf99cdeefe90a22370e1d6f5e6495.png)

### number_of_tmp_files

- 也可以通过`number_of_tmp_files`中看到是否使用了临时文件。`number_of_tmp_files`表示的是，排序过程中使用的临时文件数。这个文件数可能是几十个，这是因为**当内存放不下时，就需要使用外部排序，一般使用归并排序算法。MySQL将需要排序的数据分成几十份，每一份单独排序后存在这些临时文件中。然后把这些有序文件再合并成一个有序的大文件。**

- 如果`sort_buffer_size`超过了需要排序的数据量的大小，`number_of_tmp_files`就是0，表示排序可以直接在内存中完成。否则就需要放在临时文件中排序。`sort_buffer_size`越小，需要分成的份数越多，`number_of_tmp_files`的值就越大。

### 其余字段

- `examined_rows=4000`表示参与排序的行数是4000行。
- `sort_mode`里面的`packed_additional_fields`的意思是，排序过程中对字符串做了“紧凑”处理。即使name字段的定义是varchar(16)，在排序过程中还是要安卓实际长度来分配空间的。
- 为了避免对结论造成干扰，请把 `internal_tmp_disk_storage_engine` 设置成 MyISAM。否则，`select @b-@a` 的结果会显示为 4001。这是因为查询 `OPTIMIZER_TRACE` 这个表时，需要用到临时表，而 `internal_tmp_disk_storage_engine` 的默认值是 InnoDB。如果使用的是 InnoDB 引擎的话，把数据从临时表取出来的时候，会让 `Innodb_rows_read` 的值加 1。

## rowid排序

```mysql
SET max_length_for_sort_data = 16;
```

- `max_length_for_sort_data`，是**MySQL中专门控制用于排序的行数据的长度的一个参数。**如果单行的长度超过这个值，MySQL就认为单行太大，采用另一种算法。

### 语句执行流程：

1. 初始化`sort_buffer`，确定放入两个字段，即name和id；
2. 从索引city找到第一个满足`city='杭州'`条件的主键id；
3. 到主键id索引取出整行，取name、id这两个字段，存入`sort_buffer`中；
4. 从索引city取下一个记录的主键id；
5. 重复步骤3、4知道不满足`city='杭州'`条件为止；
6. 对`sort_buffer`中的数据按照字段name进行排序；
7. 遍历排序结果，取前1000行，并按照id的值回到原表中取出city、name和age三个字段返回给客户端。

<img src="http://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/MySQL%E5%AE%9E%E6%88%9845%E8%AE%B2/assets/dc92b67721171206a302eb679c83e86d.jpg" alt="img" style="zoom:50%;" />

- 对比全字段排序，rowid排序多访问了一次表t的主键索引，就是步骤7。
- 最后的结果集是一个逻辑概念，实际上MySQL服务端从排序后的`sort_buffer`中依次取出id，然后到原表查到city、name和age这三个字段的结果，不需要在服务端再耗费内存存储结果，是直接返回给客户端的。

![img](http://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/MySQL%E5%AE%9E%E6%88%9845%E8%AE%B2/assets/27f164804d1a4689718291be5d10f89b.png)

- 此时，rowid 排序的 `OPTIMIZER_TRACE`输出中，`examined_rows`的值还是4000，表示用于排序的数据是4000行。但是`select @b - @a`这个语句的值变成5000了。因为此时除了排序过程外，在排序完成后，还要根据id去原表取值。由于语句是limit 1000，因此会多读1000行。
- `sort_mode`变成了`<sort_key,rowid>`，表示参与排序的只有name和id两个字段。
- `number_of_tmp_files`变成10了，是因为这时候参与排序的行数虽然仍是4000行，但是每一行都变小了，因此需要排序的总量就变小了，需要的临时文件也少了。

## 全字段排序 VS rowid排序

- 如果MySQL担心排序内存太小，会影响排序效率，才会采用rowid排序算法，这样排序过程中一次可以排序更多行，但是需要再回到原表去取数据。
- 如果MySQL认为内存足够大，会优先选择全字段排序，把需要的字段都放到`sort_buffer`中，这样排序后就会直接从内存里面返回查询结果了，不用再回到原表去取数据。
- 这也就体现了 MySQL 的一个设计思想：**如果内存够，就要多利用内存，尽量减少磁盘访问。**对于InnoDB来说，rowid排序会要求回表多，造成磁盘读，因此不会被优先选择。

### 优化

- 从上述两个执行过程来看，MySQL之所以需要生成临时表，并在临时表上做排序操作，其原因是原来的数据都是无序的。所以并不是所有的order by语句，都需要排序操作。
- 通过引用覆盖索引进行的查询，也就不需要临时表和排序了，因为查询出的数据集本身就是有序的。这里并不是说要为了每个查询能用上覆盖索引，就要把语句中涉及的字段都建上联合索引，毕竟索引还是有维护代价的。这是一个需要权衡的决定。
- **覆盖索引是指，索引上的信息足够满足查询请求，不需要再回到主键索引上去取数据。**
- 对于类似`city in ('杭州', '苏州')`这种语句，可以**对应字段分别查询，在代码层面做归并处理。**

# MySQL的随机逻辑

```mysql
-- 新建word表，里面插入了10000行记录
mysql> select word form words order by rand() limit 3;
```

这个语句的意思很直白，随机排序取前3个。虽然这个SQL语句写法很简单，但执行流程却有些复杂。使用explain命令查看语句的执行情况。

![img](http://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/MySQL%E5%AE%9E%E6%88%9845%E8%AE%B2/assets/59a4fb0165b7ce1184e41f2d061ce350.png)

- Extra字段显示`Using temporary`，表示使用的是临时表；
- `Using filesort`表示的是需要执行排序操作；
- 这个Extra的意思是**需要临时表，并且需要在临时表上排序。**

## 随机排序流程

- 对于InnoDB表来说，执行**全字段排序**会减少磁盘访问，因此会被优先选择。
- 对于内存表，回表过程只是简单地根据数据行的位置，直接访问内存得到的数据，根本不会导致多访问磁盘。
- 优化器没有了这一层顾虑，那么它会优先考虑的，就是用于排序的行越小越好了，所以，MySQL这时就会选择**rowid排序**。

### 执行步骤：

1. 创建一个临时表。这个临时表使用的是memory引擎，表里有两个字段，第一个字段是`double`类型，记为字段R，第二个字段是`varchar(64)`类型，记为字段W。并且，这个表没有建索引。
2. 从words表中，按主键顺序取出所有的word值。对于每一个word，调用rand()函数生成一个大于0小于1的随机小数，并把这个随机小数和word分别存入临时表的R和W字段中，到此，扫描行数是10000。
3. 现在临时表有10000行数据了，接下来你要在这个没有索引的内存临时表上，按照字段R排序。
4. 初始化`sort_buffer`。`soft_buffer`中有两个字段，一个是double类型，另一个是整型。
5. 从内存临时表中一行一行地取出R值和位置信息，分别存入`sort_buffer`中的两个字段里。这个过程要对内存临时表做全表扫描，此时扫描行数增加10000，变成了20000。
6. 在`sort_buffer`中根据R的值进行排序。注意，这个过程没有涉及到表操作，所以不会增加扫描行数。
7. 排序完成后，取出前三个结果的位置信息，依次到内存临时表汇总取出word值，返回给客户端。这个过程中，访问了表的三行数据，总扫描行数变成了20003。

<img src="http://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/MySQL%E5%AE%9E%E6%88%9845%E8%AE%B2/assets/2abe849faa7dcad0189b61238b849ffc.png" alt="img" style="zoom:50%;" />

### MySQL定位表数据

- rowid实际上它表示的是，每个引擎用来唯一标识数据行的信息，也就是pos数据。
- 对于有主键的InnoDB来说，这个rowid就是主键ID；
- 对于没有主键的InnoDB表来说，这个rowid就是由系统生成的；
- MEMORY引擎不是索引组织表，实际上，rowid其实就是数组的下标。

## 磁盘临时表

- `tmp_table_size`这个配置限制了内存临时表的大小，默认值是16M。如果临时表大小超过了`tmp_table_size`，那么内存临时表就会转成磁盘临时表。
- 磁盘临时表使用的引擎默认是InnoDB，是由参数`internal_tmp_disk_storage_engine`控制的。当使用磁盘临时表的时候，对应的就是**一个没有显式索引的InnoDB的表的排序过程**。
- 当查询的行数超过了设置的`sort_buffer_size`大小，就只能使用归并排序算法。否则，使用优先队列算法。

## 随机排序方法

### 随机算法1

1. 取得这个表的主键id的最大值M和最小值N；
2. 用随机函数生成一个最大值到最小值之间的数$X = (M-N)*rand() + N$；
3. 取不小于X的第一个ID的行。

```sql
mysql> select max(id),min(id) into @M,@N from t ;
set @X= floor((@M-@N+1)*rand() + @N);
select * from t where id >= @X limit 1;
```

这个方法效率很高，因为取max(id)和min(id)都是不需要扫描索引的，而第三步的select也可以用索引快速定位，可以认为就只扫描了3行。但实际上，这个算法本身得出的ID中间可能有空洞，因此选择不同行的概率不一样，不是真正的随机。

### 随机算法2

1. 取得整个表的行数，并记为C。
2. 取得$Y = floor(C * rand())$。floor函数在这里的作用，就是取整部分。
3. 再用`limit Y, 1`取得一行。

```sql
mysql> select count(*) into @C from t;
set @Y = floor(@C * rand());
set @sql = concat("select * from t limit ", @Y, ",1");
prepare stmt from @sql;
execute stmt;
DEALLOCATE prepare stmt;
```

由于limit后面的参数不能直接跟变量，所以代码使用了`perpare+execute`的方法。该算法解决了算法1里面明显的概率不均匀的问题。

MySQL处理`limit Y,1`的做法就是按顺序一个一个地读出来，丢掉前Y个，然后把下一个记录作为返回结果，因此这一步需要扫描`Y+1`行。再加上，第一步扫描的C行，总共需要扫描`C+Y+1`行，执行代价比算法1的代价要高。

### 随机算法3

1. 取得整个表的行数，记为C；
2. 根据相同的随机方法得到Y1、Y2、Y3；
3. 在执行三个`limit Y,1`语句得到三行数据。

```sql
mysql> select count(*) into @C from t;
set @Y1 = floor(@C * rand());
set @Y2 = floor(@C * rand());
set @Y3 = floor(@C * rand());
select * from t limit @Y1，1； // 在应用代码里面取 Y1、Y2、Y3 值，拼出 SQL 后执行
select * from t limit @Y2，1；
select * from t limit @Y3，1；
```

# SQL语句性能分析

```sql
mysql> CREATE TABLE `tradelog` (
  `id` int(11) NOT NULL,
  `tradeid` varchar(32) DEFAULT NULL,
  `operator` int(11) DEFAULT NULL,
  `t_modified` datetime DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `tradeid` (`tradeid`),
  KEY `t_modified` (`t_modified`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

## 条件字段函数操作

对于上述表，统计发生在所有年份中7月份的交易记录总数：

```sql
mysql> select count(*) from tradelog where month(t_modified)=7;
```

- 虽然`t_modified`字段上有索引，但是使用函数计算的时候，就用不上索引了。
- 对索引字段做函数操作，可能会破坏索引值的有序性，因此优化器就决定**放弃走树搜索功能**。
- 类似的，对于条件`where id - 1 = 1000`也要改写成`where id = 1000 + 1`，这样优化器才会快速定位到9999。

## 隐式类型转换

```sql
mysql> select * from tradelog where tradeid=110717;
```

- tradeid的字段类型是varchar(32)，而输入的参数却是整型，所以做了类型转换。
- 类型转换相当于做了函数操作，所以MySQL会走全表扫描。
- 在MySQL里面，字符串和数字作比较的话，是将字符串转换成数字。

## 隐式字符编码转换

- 当两个表的字符集不同，一个是utf8，一个是utf8mb4，所以做表连接的时候用不上关联字段的索引。
- 同样地，字符集utf8mb4是utf8的超集，所以要把utf8字符串转成utf8mb4字符集，再做比较。实际上也是用来转换函数操作。

#### 优化例子

tradelog表是utf8mb4，trade_detail表是utf8字符集，对该查询进行优化。

```sql
select d.* from tradelog l, trade_detail d where d.tradeid=l.tradeid and l.id=2;
```

将trade_detail表上的tradeid字段的字符集也改成utf8mb4，这样就没有字符集转换问题。

```sql
alter table trade_detail modify tradeid varchar(32) CHARACTER SET utf8mb4 default null;
```

如果数据量比较大，或者业务上暂时无法做这个DDL，可以采用修改SQL语句的方法。

```sql
mysql> select d.* from tradelog l , trade_detail d 
where d.tradeid=
CONVERT(l.tradeid USING utf8) and l.id=2; 
```

#### 字符串检索场景

```sql
-- 建表
mysql> CREATE TABLE `table_a` (
  `id` int(11) NOT NULL,
  `b` varchar(10) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `b` (`b`)
) ENGINE=InnoDB;
-- 假设现在表里面，有 100 万行数据，其中有 10 万行数据的 b 的值是’1234567890’
mysql> select * from table_a where b='1234567890abcd';
```

- 执行流程：

1. 在传给引擎执行的时候，做了字符截断。因为引擎里面这个行只定义了长度是 10，所以只截了前 10 个字节，就是’1234567890’进去做匹配；
2. 这样满足条件的数据有 10 万行；
3. 因为是 select *， 所以要做 10 万次回表；
4. 但是每次回表以后查出整行，到 server 层一判断，b 的值都不是’1234567890abcd’;
5. 返回结果是空。

- SQL虽然执行过程中可能经过函数操作，但是最终在拿到结果后，server 层还是要做一轮判断的。
