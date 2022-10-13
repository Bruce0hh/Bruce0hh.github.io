[TOC]

# Redis 持久化

## RDB

### rdb的创建

> 有两个Redis命令可以用于生成RDB文件，一个是SAVE，另一个是BGSAVE。
> 
- 创建`rdb`文件的实际工作是由`rdbSave`函数完成。`save`和`bgsave`以不同的方式调用这个函数。
- `save`通过直接使用`rdbSave`，创建`rdb`文件。
- `bgsave`创建子进程，子进程通过调用函数创建`rdb`文件，然后向父进程发送信号。
- `save`会阻塞服务器，`bgsave`则不会。

### rdb的更新

- `aof`文件的更新频率比`rdb`文件的更新频率高。
- 如果服务器开启了`aof`，就优先使用`aof`；只有`aof`处于关闭状态，才会使用`rdb`文件。
- `save`命令执行时，`Redis`服务器会被阻塞，客户端所有命令请求都会被拒绝。
- 服务器在载入`rdb`文件时，会一直处于阻塞，直到载入完成。

### BGSAVE

1. `bgsave`执行期间，客户端的`save`命令会被拒绝。这是为了避免服务器进程和子进程同时执行两个`rdbsave`调用，从而产生竞争条件。
2. `bgsave`执行期间，客户端的`bgsave`命令会被拒绝。因为同时执行两个`bgsave`也会产生竞争条件。
3. `bgrewriteaof`和`bgsave`两个命令不能同时执行：
    - 若`bgsave`正在执行，`bgrewriteaof`会延迟到`bgsave`执行完毕后执行。
    - 若`bgrewriteaof`正在执行，那么`bgsave`会被拒绝。
    - 虽然两者都是由子进程执行，没有并发问题；但两个进程都执行大量I/O操作，不能同时执行时为了性能考虑。

### 自动保存

> 当Redis服务器启动时，用户可以通过指定配置文件或者传入启动参数的方式设置save选项，如果用户没有主动设置save选项，那么服务器会为save选项设置默认条件。save选项：save [seconds] [changes]
> 

### redisServer数据结构

- `saveparam *saveparam`：记录保存条件的数组。
- `long dirty`：修改计数器。
- `time_t lastsave`：上一次执行保存的时间。

**saveparam**

- `time_t seconds`：秒数。
- `int changes`：修改数。

**dirty & lastsave**

- `dirty`计数器记录距离上一次成功执行`save`或`bgsave`后，服务器对所有数据库进行了多少次修改（CRUD）。
- `lastsave`记录了上一次成功执行`save`或`bgsave`命令的时间。

**serverCron**

- 使用`serverCron`操作函数判断检查条件是否满足。
- 遍历`saveparams`数组中所有保存条件，只要有一个条件被满足，服务器就会执行`bgsave`命令。

### RDB文件结构

### rdb文件数据结构

- `REDIS`：五个字符，判断是否为RDB文件。
- `db_version`：记录RDB文件的版本号。
- `database`：包含多个数据库，以及其中的键值对数据。一个数据库对应一个database文件。若数据库为空，没有数据，那么这部分也为空，长度为0。
- `EOF`：EOF常量长度为1。标志正文结束，所有数据库的键值对都已经载入完毕。
- `check_num`：一个8字节长的无符号整数，保存一个校验和，由上述四个部分计算得出。

### database数据结构

- `SELECTDB`：SELECTDB常量长度为1字节。标志接下来要读入一个数据库号码。
- `db_number`：读入db_number部分之后，服务器会调用`SELECT`命令，根据读入的数据库号码进行数据库切换，使得之后的键值对可以载入到相应的数据库中。
- `key_value_pairs`：保存了数据库中所有键值对数据，若有过期时间，也会和键值对在一起保存。

### key_value_pairs数据结构

- `EXPIRETIME_MS`：EXPIRETIME_MS常量的长度为1字节，标志接下来要读入一个过期时间。
- `ms`：一个8字节长的带符号整数，记录一个以毫秒为单位的UNIX时间戳，这个时间戳就是键值对的过期时间。
- `TYPE`：TYPE记录了value的类型，长度为1字节。一共有9种TYPE，每个都代表了一种对象类型或底层编码。程序会根据TYPE的值来决定如何读入和解释value的数据。
- `key`：key是一个字符串对象，保存了键值对的键对象。
- `value`：value是键值对的值对象。

### value的编码

1. 字符串对象
2. 列表对象
3. 集合对象
4. 哈希表对象
5. 有序集合对象
6. INSET编码的集合
7. ZIPLIST编码的列表、哈希表或有序集合

## AOF

### aof实现

### 命令追加

- 当aof开启时，服务器在执行完一个写命令之后，会以协议格式将被执行的写命令追加到`aof_buf`缓冲区末尾。

### aof写入与同步

> 服务器在处理文件事件时可能会执行写命令，使得一些内容被追加到aof_buf缓冲区里面，所以在服务器每次结束一个事件循环之前，它都会调用flushAppendOnlyFile函数。
> 
- `redis`提供了`fsync`和`fdatasync`两个同步函数，它们可以强制让操作系统立即将缓冲区中的数据写入到硬盘里面，从而确保写入数据的安全性。
- `flushAppendOnlyFile`函数的行为由服务器配置的`appendfsync`选项的值来决定。`appendfsync`默认值为`everysec`。
- `appendfsync`的值为always时，服务器在每个事件循环都要将`aof_buf`缓冲区中所有内容写入到`AOF`文件，并且同步`AOF`文件，所以`always`效率是最慢的一个，也是最安全的一个。即使出现故障停机，`AOF`持久化也只会丢失一个事件循环中所产生的命令数据。
- `appendfsync`的值为everysec时，服务器在每个事件循环都要将`aof_buf`缓冲区中所有内容写入到`AOF`文件，并每隔一秒就在子线程中对`AOF`文件进行一次同步。就算出现故障停机，数据库也只丢失一秒钟的命令数据。
- `appendfsync`的值为no时，服务器在每个事件循环都要将`aof_buf`缓冲区中所有内容写入到`AOF`文件，至于何时同步，由操作系统控制。由于该模式下，`flushAppendOnlyFile`调用无需执行同步操作，所以此时`AOF`文件写入速度是最快的。

### aof文件的载入与还原

1. 创建一个不带网络连接的伪客户端（`fake client`）。
2. 从`aof`文件中分析并读出一条写命令。
3. 使用伪客户端执行被读出的写命令。
4. 重复上述2、3，直到所有写命令都被处理完毕为止。

### aof文件的重写

> AOF重写可以产生一个新的AOF文件，这个新的AOF文件和原有的AOF文件所保存的数据库状态一样，但体积更小。
> 
1. 在执行`BGREWRITERAOF`命令时，服务器会维护一个`AOF`重写缓冲区，该缓冲区会在子进程创建新`AOF`文件期间，记录服务器执行的所有写命令。
2. 当子进程完成创建新`AOF`文件的工作之后，服务器会将重写缓冲区中的所有内容追加到新`AOF`文件的末尾，使得新旧俩文件状态一致。
3. 最后，服务器使用新`AOF`文件替换旧文件，完成`AOF`文件重写操作。