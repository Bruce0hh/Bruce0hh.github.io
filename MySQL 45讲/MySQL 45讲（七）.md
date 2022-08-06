# 查单行语句的SQL性能

## 查询长时间不返回

执行`show processlist`命令，查看当前语句处于什么状态。

### 等MDL锁

- `show processlist`命令查看，status显示`Waiting for table metadata lock`，这个状态表示的是，现在有一个线程正在表T上请求或者持有MDL写锁，把select语句堵住了。
- 通过`sys.schema_table_lock_watis`这张表，我们就可以直接找出造成阻塞的`process id`，把这个连接用kill命令断开即可。

### 等flush

```sql
mysql> select * from information_schema.processlist where id=1;
```

- 查出来这个线程的状态是`Waiting for table flush`，这个状态表示的是，现在有一个线程正要对表t做flush操作。

### 等行锁

```sql
mysql> select * from t where id=1 lock in share mode; 
-- 查询写锁位置
mysql> select * from t sys.innodb_lock_waits where locked_table='`test`.`t`'\G
```

<img src="http://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/MySQL%E5%AE%9E%E6%88%9845%E8%AE%B2/assets/d8603aeb4eaad3326699c13c46379118.png" alt="img" style="zoom: 67%;" />

- `blocking_pid`表示阻塞的进程号是4；
- 使用`kill query 4`，这个命令表示停止4号线程当前正在执行的语句，由于占有行锁的是update语句，语句是之前完成的，现在执行`kill query`，无法让事务去掉行锁。所以，**这个命令不能达到释放锁的效果。**
- `kill 4`这个命令是有效的，也就是说直接断开这个连接。连接被断开的时候，会自动回滚这个连接里面正在执行的线程，也就释放了行锁。

## 查询慢

- 坏查询不一定是慢查询。

```sql
mysql> select * from t where id=1;                         -- A
mysql> select * from t where id=1 lock in share mode;       -- B
```

- 在事务1启动后，事务2执行update语句，此时如果事务1执行语句A，此时是一致性读，如果执行语句B，那就是快照读。
- 所以造成一个情况，虽然语句B加锁，但是查询速度更快，因为A要从当前数值重新通过undo log计算之前视图的值。

# 幻读

```sql
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `c` int(11) DEFAULT NULL,
  `d` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `c` (`c`)
) ENGINE=InnoDB;

insert into t values(0,0,0),(5,5,5),
(10,10,10),(15,15,15),(20,20,20),(25,25,25);
```

> 以下场景未特别说明的，均为RR隔离级别，且场景均为理论假设，实际结果自行验证

## 幻读是什么

<img src="http://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/MySQL%E5%AE%9E%E6%88%9845%E8%AE%B2/assets/5bc506e5884d21844126d26bbe6fa68b.png" alt="img" style="zoom: 67%;" />

1. 在可重复读隔离级别下，普通的查询是快照读，是不会看到别的事务插入的数据的。因此，**幻读在“当前读”下才会出现。**
2. 上面 session B 的修改结果，被 session A 之后的 select 语句用“当前读”看到，不能称为幻读。**幻读仅专指“新插入的行”。**

## 幻读造成的问题

1. **语义被破坏。**
2. **造成数据不一致。**

<img src="http://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/MySQL%E5%AE%9E%E6%88%9845%E8%AE%B2/assets/7a9ffa90ac3cc78db6a51ff9b9075607.png" alt="img" style="zoom:67%;" />

T1处，session A只是给`id=5`这一行加了行锁，没有给`id=0`这行加上锁。但是由于在session B的更新，破坏了session A里语句中要锁住所有`d=5`的行的加锁声明。session C同理。

<img src="http://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/MySQL%E5%AE%9E%E6%88%9845%E8%AE%B2/assets/dcea7845ff0bdbee2622bf3c67d31d92.png" alt="img" style="zoom:67%;" />

> 锁的设计是为了保证数据的一致性。而这个一致性，不止是数据库内部数据状态在此刻的一致性，还包含了数据和日志在逻辑上的一致性。

- update 的加锁语义和 `select …for update` 是一致的。session A 声明说“要给 `d=5` 的语句加上锁”，就是为了要更新数据，新加的这条 update 语句就是把它认为加上了锁的这一行的 d 值修改成了 100。

- 语句分析：
  
  1. 经过 T1 时刻，id=5 这一行变成 (5,5,100)，当然这个结果最终是在 T6 时刻正式提交的 ;
  2. 经过 T2 时刻，id=0 这一行变成 (0,5,5);
  3. 经过 T4 时刻，表里面多了一行 (1,5,5);
  4. 其他行跟这个执行序列无关，保持不变。

- binlog 分析：
  
  1. T2 时刻，session B 事务提交，写入了两条语句；
  2. T4 时刻，session C 事务提交，写入了两条语句；
  3. T6 时刻，session A 事务提交，写入了 update t set d=100 where d=5 这条语句。

- 从上述分析发现，**binlog 的语句序列执行是和事务执行最后的结果是不一致的。如果将所有的记录都加上锁，实际上也无法解决不一致的问题。因为`id=1`这行是后来插入的，不存在也就加不了锁。**

## 解决幻读

> 产生幻读的原因是，行锁只能锁住行，但是新插入记录这个动作，要更新的是记录之间的“间隙”。因此，为了解决幻读问题，InnoDB 只好引入新的锁，也就是间隙锁 (Gap Lock)。

- 间隙锁和行锁合称 next-key lock，每个 next-key lock 是前开后闭区间，间隙锁是开区间。实际上，InnoDB给每个索引加了一个不存在的最大值`superemum`，使得每个间隙锁都是开区间。
- 比如，当执行`select * from t where d = 5 for update`的时候，就不止是给数据库中已有的6个记录加上锁，同时还加上了7个间隙锁。这样就确保了无法再插入新的记录。
- 也就是说，在一行行的扫描过程中，不仅将给行加上了行锁，还给行两边的空隙，也加上了间隙锁。
- **数据行是可以加上锁的实体，数据行之间的空隙，也是可以加上锁的实体。行锁是互相之间有冲突关系，但是间隙锁之间没有冲突关系，和间隙锁有冲突关系的是“往这个间隙中插入一个记录”。**
- 当两个事务的给同一范围的数据行加上间隙锁，**虽然两个间隙锁之间不会冲突，但是他们的DDL语句会互相与对方的间隙锁冲突**。

# Next-key Lock

> InnoDB引擎的`next-key lock`是通过在索引上加锁实现的。

## 加锁规则

1. **原则1：**加锁的基本单位是`next-key lock`。`next-key lock`的区间是前开后闭。
2. **原则2：**查找过程中访问到的对象才会加锁。
3. **优化1：**索引上的等值查询，给**唯一索引**加锁的时候，`next-key lock`退化为行锁。
4. **优化2：**索引上的等值查询，向右遍历时且**最后一个不满足等值条件**的时候，`next-key lock`退化为间隙锁。
5. **bug：**唯一索引上的范围查询会**访问到不满足条件的第一个值**为止。

## 案例

```sql
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `c` int(11) DEFAULT NULL,
  `d` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `c` (`c`)
) ENGINE=InnoDB;

insert into t values(0,0,0),(5,5,5),
(10,10,10),(15,15,15),(20,20,20),(25,25,25);
```

### 案例一：等值查询间隙锁

<img src="http://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/MySQL%E5%AE%9E%E6%88%9845%E8%AE%B2/assets/585dfa8d0dd71171a6fa16bed4ba816c.png" alt="img" style="zoom:67%;" />

1. 加锁单位是`next-key lock`，因此会给(5,10]加上`next-key lock`。
2. 等值查询`id=7`,`id=10`不满足条件，`next-key lock`退化成间隙锁，所以最后加锁的范围是(5,10)。

### 案例二：非唯一索引等值锁

<img src="http://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/MySQL%E5%AE%9E%E6%88%9845%E8%AE%B2/assets/465990fe8f6b418ca3f9992bd1bb5465.png" alt="img" style="zoom: 50%;" />

1. 加锁单位是`next-key lock`,所以加锁的范围是(0,5]。
2. 按照原则2，只有访问到的对象才会加锁，但c是普通索引，需要继续遍历到`c=10`才能放弃，所以要给(5,10]加上`next-key lock`。
3. 查询使用的是覆盖索引，并不需要访问主键索引，所以主键索引上没有任何锁。这就是为什么session B可以执行完成，并且证明了**锁是加在索引上面的**。
4. c是普通索引，所以不符合优化1的规则。但是符合优化2：等值判断，向右遍历，`next-key lock`退化成间隙锁(5,10)。

> 如果你要用 lock in share mode 来给行加读锁避免数据被更新的话，就必须得绕过覆盖索引的优化，在查询字段中加入索引中不存在的字段。

### 案例三：主键索引范围锁

<img src="http://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/MySQL%E5%AE%9E%E6%88%9845%E8%AE%B2/assets/30b839bf941f109b04f1a36c302aea80.png" alt="img" style="zoom: 50%;" />

1. 根据原则1，加上`next-key lock`(5,10]。由于是唯一索引的等值查询，使用优化1，`next-key lock`退化成行锁。
2. 范围查询继续向后查找，加上`next-key lock`(10,15]。

> 需要注意一点的是，session A查找`id=10`的时候，是当做等值查询来判断的；向右扫描到`id=15`的时候，用的是范围查询。

### 案例四：非唯一索引范围锁

<img src="http://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/MySQL%E5%AE%9E%E6%88%9845%E8%AE%B2/assets/7381475e9e951628c9fc907f5a57697a.png" alt="img" style="zoom:50%;" />

1. 根据原则1，加上`next-key lock`(5,10]和(10,15]。c不是唯一索引并且10也非不等值，所以不符合任何一个优化规则。
2. 这里需要扫描到`c=15`才停止扫描，因为InnoDB要扫到`c=15`，才知道不需要继续往后找了。

### 案例五：唯一索引范围锁bug

<img src="http://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/MySQL%E5%AE%9E%E6%88%9845%E8%AE%B2/assets/b105f8c4633e8d3a84e6422b1b1a316d.png" alt="img" style="zoom:50%;" />

1. 根据原则1，加上`next-key lock`(10,15]。
2. 但实际上，InnoDB会往前扫描到第一个不满足条件的行为止，也就是`id=20`。而且由于这是个范围扫描，因此(15,20]这个`next-key lock`也会被锁上。这就是bug的现象。

### 实例六：非唯一索引上存在“等值”的例子

```sql
mysql> insert into t values(30,10,30);
```

<img src="http://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/MySQL%E5%AE%9E%E6%88%9845%E8%AE%B2/assets/b55fb0a1cac3500b60e1cf9779d2da78.png" alt="img" style="zoom:50%;" />

1. 根据原则1，加上`next-key lock`(5,10]。
2. 但实际上，根据优化2，还会继续扫描到`c=15`这一行为止，然后退化成间隙锁(5,15)。

<img src="http://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/MySQL%E5%AE%9E%E6%88%9845%E8%AE%B2/assets/bb0ad92483d71f0dcaeeef278f89cb24.png" alt="img" style="zoom:50%;" />

### 案例七：limit语句加锁（对比案例六）

<img src="http://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/MySQL%E5%AE%9E%E6%88%9845%E8%AE%B2/assets/afc3a08ae7a254b3251e41b2a6dae02e.png" alt="img" style="zoom:50%;" />

1. 同案例六，实际上`c=10`在表中只有两条，在扫描到两条符合条件的行之后，就停止了扫描。所以加锁的范围就是(5,10]这个区间。
2. 上图session B就未被阻塞。所以，**在删除数据的时候尽量加limit。这样不仅可以控制删除数据的条数，让操作更安全，还可以减少加锁的范围。**

### 案例八：死锁

<img src="http://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/MySQL%E5%AE%9E%E6%88%9845%E8%AE%B2/assets/7b911a4c995706e8aa2dd96ff0f36506.png" alt="img" style="zoom:50%;" />

1. 根据原则1和优化2，首先在索引c上加了`next-key lock`（5,10]和间隙锁(10,15)。
2. 同样的，session B的update也会给索引c上的(5,10]的`next-key lock`，此时update语句被阻塞。
3. session A插入数据，被B的间隙锁锁住，出现死锁，InnoDB让session B回滚。

> 虽然session B中update语句被堵塞了，实际上，加锁的时候先是加 (5,10) 的间隙锁，加锁成功；然后加 c=10 的行锁，这时候才被锁住的。我们分析加锁规则的时候用`next-key lock`来分析，而具体执行的时候，是要分成间隙锁和行锁两段来执行的。

### 案例九：排序语句

<img src="http://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/MySQL%E5%AE%9E%E6%88%9845%E8%AE%B2/assets/3a7578e104612a188a2d574eaa3bd81e.png" alt="img" style="zoom: 67%;" />

1. 由于是排序语句`order by c desc`，第一个要定位索引c上最右边的行`c=20`的行，所以会加上间隙锁(20,25)和`next-key lock`(15,20]。
2. 在索引c上向左遍历，要扫描到`c=10`才停下来，所以`next-key lock`会加到(5,10]，这正是阻塞session B的insert语句的原因。
3. 在扫描过程中，`c=20`、`c=15`、`c=10` 这三行都存在值，由于是 `select *`，所以会在主键 id 上加三个行锁。
4. 所以，锁的范围就是索引c上的(5,25)和主键索引上的`id=5`、`id=20`两个行锁。

## 小结

1. 以上所有的案例都是在RR(repeatable-read)可重复读隔离级别下验证的。RR遵守两阶段锁协议，所有的加锁资源，都是在事务提交或回滚的时候释放的。
2. `next-key lock`实际上是由间隙锁和行锁实现的。如果使用RC(read-committed)读提交隔离级别的话，就更好分析了，过程中去掉间隙锁留下行锁部分即可。但实际上RC在外键的场景还是有间隙锁，相对比较复杂。
3. RC级别下，还有一个优化规则，就是在语句执行过程中加上的锁，在语句执行完成后，就要把"不满足条件的行"上的行锁直接释放了，不需要等到事务提交。也就是说，RC级别下，锁的范围更小，时间更短，这也是很多业务默认使用RC级别的原因。
4. 间隙锁不只在二级索引有，实际上，有间隙的地方就有可能有间隙锁。
5. varchar类型的加锁规则和integer是一样的，排好序后，相邻两个值之间就有间隙。

# MySQL临时性优化

## 短连接风暴

- MySQL建立连接的成本是很高的，除了网络连接外，还需要做登录权限判断和获得读写权限。
- 当数据库处理请求慢一些，短连接模型的缺点就暴露出来，连接数就会暴涨。有可能超过`max_connections`这个参数，并会拒绝连接。

### 解决方案：

1. **先处理那些占着连接但是不工作的线程。**使用`show processlist`查看连接进程，连接数过多，你可以优先断开事务外空闲太久的连接；如果这样不够，可以再考虑断开事务内空闲太久的连接。
   
   > 从数据库端主动断开连接可能是有损的，尤其是有的应用端收到这个错误后，不重新连接，而是直接用这个已经不能用的句柄重试查询。这会导致从应用端看上去，“MySQL 一直没恢复”。

2. **减少连接过程的消耗。**有的业务代码会在短时间内先大量申请数据库连接做备用，如果现在**数据库确认是被连接行为打挂了**，那么一种可能的做法，是**让数据库跳过权限验证阶段**。跳过权限验证的方法是：**重启数据库，并使用`–skip-grant-tables` 参数启动**。这样，整个 MySQL 会跳过所有的权限验证阶段，包括连接过程和语句执行过程在内。

## 慢查询性能问题

1. **索引没有设计好。**
   
   - 一般是通过紧急创建索引来解决。
   
   - 比较理想的是能够在备库先执行。假设现在是一主库A、备库B。
     
     a. 在备库B上执行`set sql_log_bin=off`，也就是不写binlog，然后执行`alter table`语句加上索引；
     
     b. 执行主备切换。
     
     c. 这时候主库是B，备库是A。在A上执行`set sql_log_bin=off`，然后执行`alter table`语句加上索引。

2. **SQL语句没有写好。**
   
   - 可以通过改写SQL语句来处理。可以通过`query_rewrite`功能，改写成另外一种模式。
   - 使用`call query_rewrite.flush_rewrite_rules()`这个存储过程，是让插入的新规则生效，也就是我们说的“查询重写”。

3. **MySQL选错了索引。**
   
   - 同样的使用查询重写，也可以给原来的语句加上`force index`，解决了选错索引的问题。

### 预防方案：

1. 上线前，在测试环境，把慢查询日志(slow log)打开，并且把`long_query_time`设置为0，确保每个语句都会被记录入慢查询日志；
2. 在测试表里插入模拟线上的数据，做一遍回归测试；
3. 观察慢查询日志里每类语句的输出，特别留意`Rows_examined`字段是否与预期一致。

## QPS突增问题

> 有时候由于业务突然出现高峰，或者应用程序 bug，导致某个语句的 QPS 突然暴涨，也可能导致 MySQL 压力过大，影响服务。

1. 如果一种是由全新业务的bug导致的。假设你的DB运维是比较规范的，也就是说白名单一个一个加的。这种情况下，如果能够确定业务方会下掉这个功能，只是时间上没那么快，那么就可以从数据库端直接把白名单去掉。
2. 如果这个新功能使用的是单独的数据库用户，可以用管理员账号把这个用户删掉，然后断开现有连接。这样，这个新功能的连接不成功，由它引发的 QPS 就会变成 0。
3. 如果这个新增的功能跟主体功能是部署在一起的，那么我们只能通过处理语句来限制。这时，我们可以使用上面提到的查询重写功能，把压力最大的 SQL 语句直接重写成`select 1`返回。
   - 如果别的功能里面也用到了这个SQL语句模板，会有误伤；
   - 很多业务并不是靠这个语句就能完成逻辑的，所以如果单独把这一个语句以`select 1`的结果返回的话，可能会导致后面的业务逻辑一起失败。
