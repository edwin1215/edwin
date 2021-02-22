# Redis

## 一、数据结构与对象



### 1 动态字符串





### 2 链表





### 3 字典





### 4 跳跃表





### 5 整数集合





### 6 压缩列表





### 7 对象







## 二、单机数据库的实现



### 1 数据库



#### 1.1 清理过期键策略

- 定时删除
  - 优点：对内存十分友好，可以保证过期键及时被清理，并释放占用内存
  - 缺点：对CPU最不友好，体现在两方面：
    - 在过期键比较多的情况下，删除过期键十分占用CPU资源，将CPU时间占用在删除与任务无关的过期键上，会影响服务器的响应时间和吞吐量
    - 需要对每一个过期键建立timer，以维护其过期时间。
- 惰性删除
  - 优点：对CPU是最友好的
  - 缺点：对内存最不友好，会造成内存泄漏
- 定期删除
  - 综合以上两点的优缺点



#### 1.2 redis过期键删除策略

- 惰性删除
  - 所有的读写操作每次执行前都会调用**`expireIfNeeded`**函数判断键是否过期，过期则删除
- 定期删除
  - 通过**`activeExpireCycle`**函数依次遍历所有数据库的`expire`字典，每次随机取出一部分键，删除其中的过期键。一个周期结束后记录执行进度，下一次执行时接着上一次的进度进行处理。
  - 是在**`serverCron`**函数执行时调用的，**`serverCron`**函数每隔`100ms`执行一次。



#### 1.3 AOF、RDB和复制功能对过期键的处理

- ###### RDB

  - ###### 生成RDB文件：

    在执行`SAVE`或`BGSAVE`命令时，会生成RDB文件，生成RDB文件时不会保存已经失效key

  - ###### 载入RDB文件：

    在启动`Redis`服务的时候，如果开启了`RDB`功能，服务器将对RDB文件进行载入

    - 服务器为主服务器时，载入RDB文件时，程序会对文件中保存的key进行检查，已过期的key不会被载入到数据库中
    - 服务器为从服务器时，载入RDB文件时，程序会载入所有的key，不论是否过期


- ###### AOF

  - ###### AOF文件写入

    - 当键过期时，并没有被惰性删除和定期删除，AOF不会产生任何行为。

  - ###### AOF重写

    - AOF重写时不会保存已失效的Key

- ###### 复制

  3.2之前的版本，从服务器的过期键不会自动失效，在从服务器上查询请求过期键，不会删除，而是像未过期键一样处理，主服务器正常。从服务器必须等待主服务器发送DEL请求后删除。3.2之后修复此问题。



#### 1.4 数据库通知

默认状态通知功能是处于关闭状态的，因为开启**`键空间通知`**和**`键事件通知`**功能需要消耗一些 CPU性能

可以通过修改 `redis.conf` 文件， 或者直接使用 `CONFIG SET` 命令来开启或关闭键空间通知功能：

```shell
CONFIG SET notify-keyspace-events AKE	#开启字符
```

| 字符   | 发送的通知                                    |
| ---- | ---------------------------------------- |
| `K`  | 键空间通知，所有通知以 `__keyspace@<db>__` 为前缀      |
| `E`  | 键事件通知，所有通知以 `__keyevent@<db>__` 为前缀      |
| `g`  | `DEL` 、 `EXPIRE` 、 `RENAME` 等类型无关的通用命令的通知 |
| `$`  | 字符串命令的通知                                 |
| `l`  | 列表命令的通知                                  |
| `s`  | 集合命令的通知                                  |
| `h`  | 哈希命令的通知                                  |
| `z`  | 有序集合命令的通知                                |
| `x`  | 过期事件：每当有过期键被删除时发送                        |
| `e`  | 驱逐(evict)事件：每当有键因为 `maxmemory` 政策而被删除时发送 |
| `A`  | 参数 `g$lshzxe` 的别名                        |

输入的参数中至少要有一个 `K` 或者 `E` ， 否则的话， 不管其余的参数是什么， 都不会有任何通知被分发。

- 键空间通知

```shell
SUBSCRIBE __keyspace@0__:[mykey]	#键名
```

- 键事件通知

```shell
SUBSCRIBE __keyevent@0__:[del]		#事件DEL、SET、EXPIRE等
```



### 2 持久化方式



#### 2.1 RDB持久化



##### 2.1.1 RDB文件的创建

有两个Redis命令可以用于生成RDB文件，一个是**`SAVE`**，另一个是**`BGSAVE`**。

通过**`rdb.c/rdbSave`**函数创建RDB文件。



##### 2.1.2 RDB文件的载入

服务器启动时自动执行，通过**`rdb.c/rdbLoad`**函数完成。如果服务器开启了`AOF`持久化功能，那么服务器会优先使用AOF文件来还原数据库状态。



##### 2.1.3 创建和载入时的服务器状态

- **SAVE**

  阻塞Redis服务器进程，直到RDB文件生成完毕。期间服务器不能处理任何命令请求。

- **BGSAVE**

  fork出一个子进程，由子进程负责创建RDB文件。期间服务器(父进程)正常处理命令请求。

  `BGSAVE`命令和`BGREWRITEAOF`命令不能同时执行

  如果`BGSAVE`命令正在执行，那么客户端发送的`BGREWRITEAOF`命令会被延迟到`BGSAVE`命令执行完后执行。

  如果`BGREWRITEAOF`命令正在执行，那么客户端发送的`BGSAVE`命令会被服务器直接拒绝。

- **RDB文件载入**

  阻塞Redis服务器进程，直到载入RDB文件工作完成。期间服务器不能处理任何命令请求。





##### 2.1.4 自动间隔性保存

通过`save`选项来设置保存条件，可以设置多个条件，只要有一个条件满足，`BGSAVE`命令就会被执行。

```shell
save 900 1		#服务器在900秒内，对数据库进行了至少1次的修改
save 300 10		#服务器在300秒内，对数据库进行了至少10次的修改
save 60 10000	#服务器在60秒内，对数据库进行了至少10000次的修改

```

Redis服务器结构体`redisServer`中自动保存相关的属性：

| 属性         | 含义                                       |
| ---------- | ---------------------------------------- |
| saveparams | 一个数组，保存了`save`选项设置的保存条件。                 |
| dirty      | 计数器，记录距上一次`SAVE`或`BGSAVE`命令之后，服务器对数据库状态(所有数据库)进行了对少次修改 |
| lastsave   | UNIX时间戳，记录上一次`SAVE`或`BGSAVE`成功执行的时间      |

Redis服务器周期性的操作函数`serverCron`，默认每`100ms`执行一次，每次执行都会检查`save`选项设置的条件是否满足，如果满足，则执行`BGSAVE`命令。



##### 2.1.5 RDB文件结构

- 魔数：`REDIS`，5字节。
- 数据库版本：`db_version`，4字节。
- 数据：服务器所有数据库的所有数据项。
- 结束符：`EOF`，1字节。
- 校验和：`check_num`，8字节无符号数。

通过`rdbcompression`选项来设置是否开启压缩RDB文件。

​	

#### 2.2 AOF持久化



##### 2.2.1 AOF文件

与RDB持久化通过保存数据库中键值对来记录状态不同，AOF是通过保存Redis服务器所执行的写命令来记录数据库状态的。



服务器在启动时，可以通过载入和执行AOF文件中保存的命令记录来还原数据库状态。



##### 2.2.2 AOF持久化的实现

AOF持久化的实现可以分为 命令追加`append`、文件写入`write`、文件同步`sync` 三个步骤。

###### 命令追加

当AOF持久化功能处于打开状态时，服务器执行完一条命令语句后，会以协议格式将命令追加到服务器状态的`aof_buf`缓冲区末尾。`aof_buf`是Redis服务结构体`redisServer`的一个属性。



###### 文件写入与同步

Redis服务器进程就是一个`事件循环(loop)`，这个循环又分为文件事件和时间事件：

- 文件事件：负责接收、响应客户端的命令
- 时间事件：执行像`serverCron`这样需要定时执行的函数

服务器在处理`文件事件`时可能会执行写命令，使得一些内容被追加到`aof_buf`中，因此在每次事件循环结束之前，它都会调用`flushAppendOnlyFile`函数，考虑是否要将`aof_buf`中的内容写入和保存到AOF文件中。

`flushAppendOnlyFile`函数的行为由服务器配置的`appendfsync`选项值决定，默认`everysec`

| appendfsync选项值 | flushAppendOnlyFile行为                    |
| -------------- | ---------------------------------------- |
| always         | 将aof_buf所有内容写入并同步到AOF文件中                 |
| everysec       | 将aof_buf所有内容写入到AOF文件中，<br />并判断如果上一次同步的时间距离当前时间超过1秒，<br />那么再次对AOF文件进行同步。<br />同步操作由一个线程专门负责执行。 |
| no             | 将aof_buf所有内容写入到AOF文件中，但不对AOF文件进行同步       |





##### 2.2.3 AOF文件的载入和数据还原

载入流程：

- 服务器启动载入程序
- 创建一个不带网络连接的伪客户端`fack client`，使用的命令来自于AOF文件而非网络，Redis命令只能在客户端上下文中执行。伪客户端执行命令的效果和带网络连接的客户端是一样的。
- 从AOF文件中分析并读取出一条写命令，使用伪客户端执行命令。
- 一直重复执行上一步，直到所有的写命令都被执行完毕。



##### 2.2.4 AOF重写

由于AOF模式是通过保存写操作命令来记录数据库状态的，所以随着运行时间的增长，AOF文件的体积会越来越大。体积过大的AOF文件会对Redis服务器，甚至宿主机造成影响。并且还原数据库状态的时间过长。

对此，Redis提供了AOF文件重写`rewrite`功能：创建一个新的AOF文件来代替现有的AOF文件。新的AOF文件所保存的数据库状态完全一致，但是新的AOF文件不会包含任何浪费空间和冗余的命令，因此会小很多。



##### 2.2.5 AOF重写的实现

AOF文件的重写不需要对现有的AOF文件进行任务读取、分析和写入操作，是通过读取当前的数据库状态来实现的。通过`aof_rewrite`函数实现。

为了避免在执行命令时造成客户端输入缓冲区溢出，重写程序在处理列表、集合、有序集合、哈希表这四种可能带有多个元素的键时，会先检查所包含的元素数量，如果元素数量超过了`redis.h/REDIS_AOF_REWRITE_ITEM_PER_CMD`常量值，那么重写程序会拆分多条执行语句，每条命令包含的最大元素数量为`REDIS_AOF_REWRITE_ITEM_PER_CMD`，`REDIS_AOF_REWRITE_ITEM_PER_CMD`默认值64。



##### 2.2.6 AOF后台重写

为了不造成主线程在重写过程中阻塞，Redis将AOF重写程序放到子进程中执行，这样做可以达到两个目的：

- 子进程执行重写程序时，服务器进程（父进程）可以正常执行命令请求。
- 子进程带有服务器进程的数据副本，使用子进程而不是子线程，可以避免在使用锁的情况下，保证数据的安全性。



另一个问题：子进程重写程序执行过程中，服务器进程还在对外提供服务，还要继续处理命令请求，导致重写后数据库状态不一致。

为了解决这个问题Redis服务器设置了`AOF重写缓冲区`，这个缓冲区在服务器创建重写子进程的同时开启，这时Redis服务器执行完一个写命令之后，会同时将这个写命令发送给AOF缓冲区和AOF重写缓冲区。如图



AOF后台重写触发时机：

​	`redisServer`会记录以下三个属性值：

| 属性                    | 含义                           |
| --------------------- | ---------------------------- |
| aof_current_size      | 当前AOF文件大小                    |
| aof_rewrite_min_size  | 执行AOF重写需要AOF文件大小达到的最小值，默认1MB |
| aof_rewrite_base_size | 最后一次 AOF重写之后，AOF文件大小         |
| aof_rewrite_perc      | 增量百分比，默认100%                 |

每次执行`serverCron`函数时都会检查：

- 没有BGSAVE命令在执行(RDB持久化)
- 没有BGREWRITEAOF命令在执行(AOF重写)
- 判断aof_current_size是否大于aof_rewrite_min_size
- 判断aof_current_size : aof_rewrite_base_size 是否大于等于 aof_rewrite_perc

如果4个条件都满足的情况下执行BGREWRITEAOF命令。



AOF后台重写流程如下：

- Redis服务器调用`BGREWRITEAOF`命令，开始后台重写；
- 服务器进程fork一个子进程执行AOF重写，重写的数据库状态是调用`BGREWRITEAOF`命令这一时刻的快照，遍历键空间、失效键空间。同时开启AOF重写缓冲区；
- 重写程序执行过程中，服务器正常处理命令请求，处理成功后会同时将这个写命令发送给`AOF缓冲区`和`AOF重写缓冲区`；
- 重写程序执行完毕后，子进程会向服务器进程发送一个信号，告知父进程执行完毕；
- 服务器进程接收到子进程执行完毕的信号后，会调用一个信号处理函数，并执行以下工作（**同步阻塞执行**）：
  - 将AOF重写缓冲区中的内容写入到新的AOF文件中，这是新的AOF文件和服务器当前的数据库状态一致；
  - 对新的AOF文件进行改名，原子地覆盖现有的AOF文件
- 取消同步，完成后台重写。

整个AOF后台重写的过程中，只有父进程执行信号处理函数时会造成阻塞，将AOF重写的影响降至最低。



### 3 事件



Redis服务器是一个事件驱动程序，服务器需要处理以下两类事件：

- ###### 文件事件（file event）

  Redis服务器通过套接字与客户端、其他Redis服务器进行连接

  而文件事件就是服务器对套接字操作的抽象

  服务器与客户端、其他Redis服务器的通信会产生相应的文件事件，而服务器则通过监听并处理这些事件来完成一系列网络通信操作

- ###### 时间事件（time event）

  Redis服务器中的一些操作，比如：`serverCron`函数，需要在给点的时间点执行

  时间事件就是服务器对这类定时操作的抽象



#### 3.1 文件事件

Redis基于`Reactor`模式开发了自己的网络事件处理器：

- 使用I/O多路复用程序来同时监听多个套接字，根据套接字当前执行的任务来关联不同是事件处理器
- ​





#### 3.2 时间事件







## 三、多机数据库的实现



### 1 复制

命令

- 复制主服务器：SLAVEOF <IP> <PORT>


- 取消复制：SLAVEOF NO ONE




#### 1.1 旧版复制功能



##### 1.1.1 同步

- 从机向主机发送SYNC命令。
- 主机收到SYNC命令后，立刻执行BGSAVE命令，在后台生成一个RDB文件，并使用一个缓冲区记录从此刻开始的所有写命令。
- BGSAVE命令执行完毕后，主机会将RDB文件发送给从机，从机载入RDB文件将数据库状态恢复至主机执行BGSAVE命令时的状态。
- 主机将缓冲区里的写命令(命令协议格式)，发送给从机执行（阻塞状态），同步完毕。



##### 1.1.2 命令传播

主机没接收一条写命令，就会发送给从机执行。



##### 1.1.3 缺陷

同步时机：

​	1、刚启动时，初次复制

​	A1、对于初次复制时，没什么问题

​	2、断线重连时

​	A2、对于断线重连时，主机没有必要把断线之前的数据发送给从机，这会造成资源的浪费（占用CPU、内存、IO资源、网络带宽、流量，从机全量数据的载入还会造成长时间阻塞）。



#### 1.2 新版复制功能

redis2.8之后使用的是新版复制功能，使用的PSYNC命令代替了SYNC命令。

PSYNC命令具有两种模式：完整重同步、部分重同步。



##### 1.2.1 完整重同步

用于初次启动，同SYNC命令



##### 1.2.2 部分重同步

用于断线重连的情况。

三个重要组成部分：

- 主机复制偏移量和从机复制的偏移量
- 主机复制积压缓冲区
- 服务器运行ID



###### 1.2.2.1 复制偏移量

主机和从机都会维护一个复制偏移量，每次命令传播后保存一致。

如出现不一致情况，说明主从通信或者从机内部有问题。



###### 1.2.2.2 复制积压缓冲区

复制积压缓冲区是一个由主机维护的固定长度(fixed-size)的先进先出FIFO队列，默认大小1MB。

主机进行命令传播后，还会将改写命令入队到复制积压缓冲区中。



###### 1.2.2.3 服务器运行ID

- 每个redis服务器，无论主从都会有自己的运行ID
- 运行ID在服务启动时生成，由40位16进制字符组成
- 初次复制时，主机会将自己的运行ID返回给从机，从机记录在本地
- 如果断线重连后，从机向主机发送PSYNC命令时，会将本地存储的主机运行ID一并带去
- 若此运行ID和主机运行ID相同，执行部分重同步
- 若此运行ID和主机运行ID不同，执行完整重同步



###### 1.2.2.4 PSYNC命令实现

PSYNC命令调用的两种方法：

- PSYNC ? -1

  初次同步或者之前执行过SLAVEOF NO ONE命令。主机会启用完整重同步策略。

- PSYNC <runid> <offset>

  若主机运行ID等于runid，并且offset在复制积压缓冲区内，执行部分重同步

  否则完整重同步

主机接收到PSYNC命令，会有以下三种回复：

- 返回+FULLRESYNC <runid> <offset>，表示主机将于从机执行完整重同步，从机将runid记录在本地，将offset设为自己的初始偏移量。
- 返回+CONTINUE，表示主机将会与从机执行部分重同步，从机只需等待主机的同步命令即可
- 返回-ERR，表示主机版本低于2.8，不识别PSYNC命令。



##### 1.2.3 复制的实现

1. 调用SLAVEOF <ip> <port>命令
2. 建立套接字连接（从机既是服务端，又是客户端）
3. 从机发送PING，主机响应PONG（握手）
4. 身份验证（建立真正的连接）
5. 发送端口信息
6. 同步
7. 命令传播

##### 1.2.4 心跳检测




### 2 Sentinel





### 3 集群

- 创建集群：CLUSTER MEET <IP> <PORT>
- 取消集群：CLUSTER MEET NO ONE