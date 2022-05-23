# --redis学习笔记



----



# 分布式缓存

## 单点Redis的问题

* 数据丢失问题：redis是内存存储，服务重启会丢失数据
* 并发能力问题
* 故障恢复问题：如果redis服务宕机则会导致整个服务不可用需要一个故障恢复的手段
* 存储能力问题：redis基于内存存储

解决：

* 数据丢失问题：实现Redis数据持久化
* 并发能力问题：搭建主从集群，实现读写分离
* 故障恢复问题： 利用Redis哨兵，实现健康检测和自动恢复
* 存储能力问题：搭建分片集群，利用插槽机制实现动态扩容



## redis持久化

### RDB持久化

RDB全称Redis Database Backup file（Redis数据备份文件），也被叫做Redis数据快照。简单来说就是把内存中的 所有数据都记录到磁盘中。当Redis实例故障重启后，从磁盘读取快照文件，恢复数据。 快照文件称为RDB文件，默认是保存在当前运行目录

Redis停机时会执行一次RDB

RDB持久化在四种情况下会执行：

- 执行save命令
- 执行bgsave命令
- Redis停机时
- 触发RDB条件时



* save命令会导致主进程执行RDB，这个过程中其它所有命令都会被阻塞。只有在数据迁移时可能用到。
* bgsave命令执行后会开启独立进程完成RDB，主进程可以持续处理用户请求，不受影响。
* Redis停机时会执行一次save命令，实现RDB持久化。
* Redis内部有触发RDB的机制，可以在redis.conf文件中找到

```sh
# 900秒内，如果至少有1个key被修改，则执行bgsave ， 如果是save "" 则表示禁用RDB
save 900 1  
save 300 10  
save 60 10000 


# 是否压缩 ,建议不开启，压缩也会消耗cpu
rdbcompression yes

# RDB文件名称
dbfilename dump.rdb  

# 文件保存的路径目录
dir ./ 
```



#### 原理

bgsave开始时会fork主进程得到子进程，子进程共享主进程的内存数据。完成fork后读取内存数据并写入 RDB 文件。

fork采用的是copy-on-write技术：

- 当主进程执行读操作时，访问共享内存；
- 当主进程执行写操作时，则会拷贝一份数据，执行写操作。



### AOF持久化

AOF全称为Append Only File（追加文件）。Redis处理的每一个写命令都会记录在AOF文件，可以看做是命令日志文件

```sh
# 是否开启AOF功能，默认是no
appendonly yes
# AOF文件的名称
appendfilename "appendonly.aof"
    
# 表示每执行一次写命令，立即记录到AOF文件
appendfsync always 
# 写命令执行完先放入AOF缓冲区，然后表示每隔1秒将缓冲区数据写到AOF文件，是默认方案
appendfsync everysec 
# 写命令执行完先放入AOF缓冲区，由操作系统决定何时将缓冲区内容写回磁盘
appendfsync no
```

特点：

* always：同步刷盘；可靠性高，几乎不丢失数据；性能影响大
* everysec：每秒刷盘；性能适中；最多丢失1秒钟的数据
* no：操作系统控制；性能最好；可靠性差，可能丢失大量的数据



#### AOF文件重写

因为是记录命令，AOF文件会比RDB文件大的多。而且AOF会记录对同一个key的多次写操作，但只有最后一次写操作才有意义。通过执行bgrewriteaof命令，可以让AOF文件执行重写功能，用最少的命令达到相同效果。

AOF原本有三个命令，但是`set num 123 和 set num 666`都是对num的操作，第二次会覆盖第一次的值，因此第一个命令记录下来没有意义。

所以重写命令后，AOF文件内容就是：`mset name jack num 666`

Redis也会在触发阈值时自动去重写AOF文件。阈值也可以在redis.conf中配置：

```properties
# AOF文件比上次文件 增长超过多少百分比则触发重写
auto-aof-rewrite-percentage 100
# AOF文件体积最小多大以上才触发重写 
auto-aof-rewrite-min-size 64mb 
```



## RDB和AOF对比

RDB方式bgsave的基本流程？

- fork主进程得到一个子进程，共享内存空间
- 子进程读取内存数据并写入新的RDB文件
- 用新RDB文件替换旧的RDB文件

RDB会在什么时候执行？save 60 1000代表什么含义？

- 默认是服务停止时
- 代表60秒内至少执行1000次修改则触发RDB

RDB的缺点？

- RDB执行间隔时间长，两次RDB之间写入数据有丢失的风险
- fork子进程、压缩、写出RDB文件都比较耗时



持久化方式：

* RDB：定时对整个内存做快照
* AOF：记录每一次执行的命令

数据完整性：

* RDB：不完整，两次备份之间会数据丢失
* AOF：相对完整，这取决于刷盘策略

文件大小：

* RDB：会有压缩，文件体积小
* AOF：记录命令，文件体积很大

宕机恢复速度：

* RDB：很快
* AOF：慢

数据恢复优先级：

* RDB：低，因为数据完整性不如AOF
* AOF：高，数据完整性更高

系统资源占用：

* RDB：高，大量CPU和内存消耗
* AOF：低，主要是磁盘io资源，但是AOF重写时会占用大量的CPU和内存资源

使用场景：

* RDB：可以容忍数分钟的数据丢失，追求更快的速度
* AOF：对数据安全性要求较高





# redis主从集群

单节点Redis的并发能力是有上限的，要进一步提高Redis的并发能力，就需要搭建主从集群，实现读写分离。

修改配置文件：

```sh
# 绑定地址，默认是127.0.0.1，会导致只能在本地访问。修改为0.0.0.0则可以在任意IP访问
bind 0.0.0.0
# 保护模式，关闭保护模式
protected-mode no
# 数据库数量，设置为1
databases 1
```

启动Redis：

```sh
redis-server redis.conf
```

停止redis服务：

```sh
redis-cli shutdown
```

master：主节点

slave：从节点

从节点读数据，主节点写数据



## 步骤



一个master节点：

* master：文件夹：./redis1  端口号：7001

两个slave节点：

* slave1：文件夹：./redis2  端口号：7002
* slave2：文件夹：./redis3  端口号：7003



### 1. 修改redis-6.2.4/redis.conf文件，将其中的持久化模式改为默认的RDB模式，AOF保持关闭状态

```sh
# 开启RDB
# save ""
save 3600 1
save 300 100
save 60 10000

# 关闭AOF
appendonly no
```



### 2. 在redis根目录下创建三个文件夹redis1、redis2和redis3

### 3. 将根目录下的redis.conf文件拷贝到redis1、redis2和redis3目录下

### 4. 修改每个实例的端口、工作目录

   redis1：

```sh
# 工作目录
dir ./redis1/
# 端口
port 7001
```

redis2：

```sh
# 工作目录
dir ./redis2/
# 端口
port 7002
```

redis3：

```sh
# 工作目录
dir ./redis3/
# 端口
port 7003
```

### 5. 启动

```sh
# 第一个
redis-server.exe redis1/redis.conf
# 第2个
redis-server.exe redis1/redis.conf
# 第三个
```



第一个：

```sh

C:\Program Files\redis>redis-server.exe redis1/redis.conf
[20940] 22 May 14:01:24.634 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
[20940] 22 May 14:01:24.635 # Redis version=5.0.14.1, bits=64, commit=ec77f72d, modified=0, pid=20940, just started
[20940] 22 May 14:01:24.635 # Configuration loaded
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 5.0.14.1 (ec77f72d/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 7001
 |    `-._   `._    /     _.-'    |     PID: 20940
  `-._    `-._  `-./  _.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

[20940] 22 May 14:01:24.638 # Server initialized
[20940] 22 May 14:01:24.638 * DB loaded from disk: 0.000 seconds
[20940] 22 May 14:01:24.638 * Ready to accept connections

```



第二个：

```sh

C:\Program Files\redis>redis-server.exe redis2/redis.conf
[19920] 22 May 14:02:02.715 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
[19920] 22 May 14:02:02.715 # Redis version=5.0.14.1, bits=64, commit=ec77f72d, modified=0, pid=19920, just started
[19920] 22 May 14:02:02.715 # Configuration loaded
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 5.0.14.1 (ec77f72d/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 7002
 |    `-._   `._    /     _.-'    |     PID: 19920
  `-._    `-._  `-./  _.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

[19920] 22 May 14:02:02.719 # Server initialized
[19920] 22 May 14:02:02.724 * DB loaded from disk: 0.004 seconds
[19920] 22 May 14:02:02.724 * Ready to accept connections

```



第三个：

```sh

C:\Program Files\redis>redis-server.exe redis3/redis.conf
[6148] 22 May 14:02:24.078 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
[6148] 22 May 14:02:24.078 # Redis version=5.0.14.1, bits=64, commit=ec77f72d, modified=0, pid=6148, just started
[6148] 22 May 14:02:24.078 # Configuration loaded
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 5.0.14.1 (ec77f72d/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 7003
 |    `-._   `._    /     _.-'    |     PID: 6148
  `-._    `-._  `-./  _.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

[6148] 22 May 14:02:24.081 # Server initialized
[6148] 22 May 14:02:24.086 * DB loaded from disk: 0.004 seconds
[6148] 22 May 14:02:24.086 * Ready to accept connections

```



### 6. 开启主从关系

现在三个实例还没有任何关系，要配置主从可以使用replicaof 或者slaveof（5.0以前）命令。

有临时和永久两种模式：

- 修改配置文件（永久生效）

  - 在redis.conf中添加一行配置：```slaveof <masterip> <masterport>```

- 使用redis-cli客户端连接到redis服务，执行slaveof命令（重启后失效）：

  ```sh
  slaveof <masterip> <masterport>
  ```

使用redis-cli连接7002节点

```sh
C:\Users\mao>redis-cli -p 7002
127.0.0.1:7002> ping
PONG
127.0.0.1:7002> SLAVEOF 127.0.0.1 7001
OK
127.0.0.1:7002>

```

结果：

7001节点：

```sh

C:\Program Files\redis>redis-server.exe redis1/redis.conf
[20940] 22 May 14:01:24.634 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
[20940] 22 May 14:01:24.635 # Redis version=5.0.14.1, bits=64, commit=ec77f72d, modified=0, pid=20940, just started
[20940] 22 May 14:01:24.635 # Configuration loaded
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 5.0.14.1 (ec77f72d/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 7001
 |    `-._   `._    /     _.-'    |     PID: 20940
  `-._    `-._  `-./  _.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

[20940] 22 May 14:01:24.638 # Server initialized
[20940] 22 May 14:01:24.638 * DB loaded from disk: 0.000 seconds
[20940] 22 May 14:01:24.638 * Ready to accept connections
[20940] 22 May 14:07:09.568 * Replica 127.0.0.1:7002 asks for synchronization
[20940] 22 May 14:07:09.568 * Partial resynchronization not accepted: Replication ID mismatch (Replica asked for 'b24d84d6ae98c62223cc047a8eaddb443a724d1e', my replication IDs are 'fe947a1e8a2b331b807fd7bae7e1c5056ee4ef8d' and '0000000000000000000000000000000000000000')
[20940] 22 May 14:07:09.569 * Starting BGSAVE for SYNC with target: disk
[20940] 22 May 14:07:09.576 * Background saving started by pid 20812
[20940] 22 May 14:07:09.661 # fork operation complete
[20940] 22 May 14:07:09.673 * Background saving terminated with success
[20940] 22 May 14:07:09.680 * Synchronization with replica 127.0.0.1:7002 succeeded

```

7002节点：

```sh

C:\Program Files\redis>redis-server.exe redis2/redis.conf
[19920] 22 May 14:02:02.715 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
[19920] 22 May 14:02:02.715 # Redis version=5.0.14.1, bits=64, commit=ec77f72d, modified=0, pid=19920, just started
[19920] 22 May 14:02:02.715 # Configuration loaded
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 5.0.14.1 (ec77f72d/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 7002
 |    `-._   `._    /     _.-'    |     PID: 19920
  `-._    `-._  `-./  _.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

[19920] 22 May 14:02:02.719 # Server initialized
[19920] 22 May 14:02:02.724 * DB loaded from disk: 0.004 seconds
[19920] 22 May 14:02:02.724 * Ready to accept connections
[19920] 22 May 14:07:09.305 * Before turning into a replica, using my master parameters to synthesize a cached master: I may be able to synchronize with the new master with just a partial transfer.
[19920] 22 May 14:07:09.305 * REPLICAOF 127.0.0.1:7001 enabled (user request from 'id=3 addr=127.0.0.1:53188 fd=9 name= age=23 idle=0 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=42 qbuf-free=32726 obl=0 oll=0 omem=0 events=r cmd=slaveof')
[19920] 22 May 14:07:09.565 * Connecting to MASTER 127.0.0.1:7001
[19920] 22 May 14:07:09.565 * MASTER <-> REPLICA sync started
[19920] 22 May 14:07:09.567 * Non blocking connect for SYNC fired the event.
[19920] 22 May 14:07:09.567 * Master replied to PING, replication can continue...
[19920] 22 May 14:07:09.568 * Trying a partial resynchronization (request b24d84d6ae98c62223cc047a8eaddb443a724d1e:1).
[19920] 22 May 14:07:09.576 * Full resync from master: ca9b9d1126598b78d418fb7bf136e91dd3d99174:0
[19920] 22 May 14:07:09.577 * Discarding previously cached master state.
[19920] 22 May 14:07:09.680 * MASTER <-> REPLICA sync: receiving 189 bytes from master
[19920] 22 May 14:07:09.682 * MASTER <-> REPLICA sync: Flushing old data
[19920] 22 May 14:07:09.682 * MASTER <-> REPLICA sync: Loading DB in memory
[19920] 22 May 14:07:09.684 * MASTER <-> REPLICA sync: Finished with success

```



断开redis-cli和7002节点的连接，连接7003节点：

```sh
C:\Users\mao>redis-cli -p 7002
127.0.0.1:7002> ping
PONG
127.0.0.1:7002> SLAVEOF 127.0.0.1 7001
OK
127.0.0.1:7002> exit

C:\Users\mao>redis-cli -p 7003
127.0.0.1:7003> ping
PONG
127.0.0.1:7003> SLAVEOF 127.0.0.1 7001
OK
127.0.0.1:7003>
```

结果

7001节点：

```sh

C:\Program Files\redis>redis-server.exe redis1/redis.conf
[20940] 22 May 14:01:24.634 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
[20940] 22 May 14:01:24.635 # Redis version=5.0.14.1, bits=64, commit=ec77f72d, modified=0, pid=20940, just started
[20940] 22 May 14:01:24.635 # Configuration loaded
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 5.0.14.1 (ec77f72d/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 7001
 |    `-._   `._    /     _.-'    |     PID: 20940
  `-._    `-._  `-./  _.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

[20940] 22 May 14:01:24.638 # Server initialized
[20940] 22 May 14:01:24.638 * DB loaded from disk: 0.000 seconds
[20940] 22 May 14:01:24.638 * Ready to accept connections
[20940] 22 May 14:07:09.568 * Replica 127.0.0.1:7002 asks for synchronization
[20940] 22 May 14:07:09.568 * Partial resynchronization not accepted: Replication ID mismatch (Replica asked for 'b24d84d6ae98c62223cc047a8eaddb443a724d1e', my replication IDs are 'fe947a1e8a2b331b807fd7bae7e1c5056ee4ef8d' and '0000000000000000000000000000000000000000')
[20940] 22 May 14:07:09.569 * Starting BGSAVE for SYNC with target: disk
[20940] 22 May 14:07:09.576 * Background saving started by pid 20812
[20940] 22 May 14:07:09.661 # fork operation complete
[20940] 22 May 14:07:09.673 * Background saving terminated with success
[20940] 22 May 14:07:09.680 * Synchronization with replica 127.0.0.1:7002 succeeded
[20940] 22 May 14:09:48.297 * Replica 127.0.0.1:7003 asks for synchronization
[20940] 22 May 14:09:48.297 * Partial resynchronization not accepted: Replication ID mismatch (Replica asked for '98f2da59439782b787db9f2572d4a3139b9a2610', my replication IDs are 'ca9b9d1126598b78d418fb7bf136e91dd3d99174' and '0000000000000000000000000000000000000000')
[20940] 22 May 14:09:48.298 * Starting BGSAVE for SYNC with target: disk
[20940] 22 May 14:09:48.329 * Background saving started by pid 7044
[20940] 22 May 14:09:48.401 # fork operation complete
[20940] 22 May 14:09:48.416 * Background saving terminated with success
[20940] 22 May 14:09:48.424 * Synchronization with replica 127.0.0.1:7003 succeeded

```

7003节点：

```sh

C:\Program Files\redis>redis-server.exe redis3/redis.conf
[6148] 22 May 14:02:24.078 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
[6148] 22 May 14:02:24.078 # Redis version=5.0.14.1, bits=64, commit=ec77f72d, modified=0, pid=6148, just started
[6148] 22 May 14:02:24.078 # Configuration loaded
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 5.0.14.1 (ec77f72d/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 7003
 |    `-._   `._    /     _.-'    |     PID: 6148
  `-._    `-._  `-./  _.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

[6148] 22 May 14:02:24.081 # Server initialized
[6148] 22 May 14:02:24.086 * DB loaded from disk: 0.004 seconds
[6148] 22 May 14:02:24.086 * Ready to accept connections
[6148] 22 May 14:09:47.781 * Before turning into a replica, using my master parameters to synthesize a cached master: I may be able to synchronize with the new master with just a partial transfer.
[6148] 22 May 14:09:47.781 * REPLICAOF 127.0.0.1:7001 enabled (user request from 'id=3 addr=127.0.0.1:53291 fd=9 name= age=24 idle=0 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=42 qbuf-free=32726 obl=0 oll=0 omem=0 events=r cmd=slaveof')
[6148] 22 May 14:09:48.292 * Connecting to MASTER 127.0.0.1:7001
[6148] 22 May 14:09:48.292 * MASTER <-> REPLICA sync started
[6148] 22 May 14:09:48.294 * Non blocking connect for SYNC fired the event.
[6148] 22 May 14:09:48.295 * Master replied to PING, replication can continue...
[6148] 22 May 14:09:48.296 * Trying a partial resynchronization (request 98f2da59439782b787db9f2572d4a3139b9a2610:1).
[6148] 22 May 14:09:48.329 * Full resync from master: ca9b9d1126598b78d418fb7bf136e91dd3d99174:196
[6148] 22 May 14:09:48.329 * Discarding previously cached master state.
[6148] 22 May 14:09:48.424 * MASTER <-> REPLICA sync: receiving 190 bytes from master
[6148] 22 May 14:09:48.425 * MASTER <-> REPLICA sync: Flushing old data
[6148] 22 May 14:09:48.426 * MASTER <-> REPLICA sync: Loading DB in memory
[6148] 22 May 14:09:48.428 * MASTER <-> REPLICA sync: Finished with success

```



### 7. 连接 7001节点，查看集群状态

```sh
Microsoft Windows [版本 10.0.19044.1706]
(c) Microsoft Corporation。保留所有权利。

C:\Users\mao>redis-cli -p 7002
127.0.0.1:7002> ping
PONG
127.0.0.1:7002> SLAVEOF 127.0.0.1 7001
OK
127.0.0.1:7002> exit

C:\Users\mao>redis-cli -p 7003
127.0.0.1:7003> ping
PONG
127.0.0.1:7003> SLAVEOF 127.0.0.1 7001
OK
127.0.0.1:7003> exit

C:\Users\mao>redis-cli -p 7001
127.0.0.1:7001> info replication
# Replication
role:master
connected_slaves:2
slave0:ip=127.0.0.1,port=7002,state=online,offset=490,lag=1
slave1:ip=127.0.0.1,port=7003,state=online,offset=490,lag=2
master_replid:ca9b9d1126598b78d418fb7bf136e91dd3d99174
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:490
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:490
127.0.0.1:7001>
```



### 8. 测试

只有在7001这个master节点上可以执行写操作，7002和7003这两个slave节点只能执行读操作。

   ```sh
   
   C:\Users\mao>redis-cli -p 7002
   127.0.0.1:7002> ping
   PONG
   127.0.0.1:7002> SLAVEOF 127.0.0.1 7001
   OK
   127.0.0.1:7002> exit
   
   C:\Users\mao>redis-cli -p 7003
   127.0.0.1:7003> ping
   PONG
   127.0.0.1:7003> SLAVEOF 127.0.0.1 7001
   OK
   127.0.0.1:7003> exit
   
   C:\Users\mao>redis-cli -p 7001
   127.0.0.1:7001> info replication
   # Replication
   role:master
   connected_slaves:2
   slave0:ip=127.0.0.1,port=7002,state=online,offset=490,lag=1
   slave1:ip=127.0.0.1,port=7003,state=online,offset=490,lag=2
   master_replid:ca9b9d1126598b78d418fb7bf136e91dd3d99174
   master_replid2:0000000000000000000000000000000000000000
   master_repl_offset:490
   second_repl_offset:-1
   repl_backlog_active:1
   repl_backlog_size:1048576
   repl_backlog_first_byte_offset:1
   repl_backlog_histlen:490
   127.0.0.1:7001> set a 1234567
   OK
   127.0.0.1:7001> get a
   "1234567"
   127.0.0.1:7001> exit
   
   C:\Users\mao>redis-cli -p 7002
   127.0.0.1:7002> get a
   "1234567"
   127.0.0.1:7002> set b 1
   (error) READONLY You can't write against a read only replica.
   127.0.0.1:7002> exit
   
   C:\Users\mao>redis-cli -p 7003
   127.0.0.1:7003> get a
   "1234567"
   127.0.0.1:7003> set b 1
   (error) READONLY You can't write against a read only replica.
   127.0.0.1:7003> set a 2
   (error) READONLY You can't write against a read only replica.
   127.0.0.1:7003> del a
   (error) READONLY You can't write against a read only replica.
   127.0.0.1:7003> exit
   
   C:\Users\mao>redis-cli -p 7001
   127.0.0.1:7001> del a
   (integer) 1
   127.0.0.1:7001> get a
   (nil)
   127.0.0.1:7001> exit
   
   C:\Users\mao>redis-cli -p 7002
   127.0.0.1:7002> get a
   (nil)
   127.0.0.1:7002> exit
   
   C:\Users\mao>redis-cli -p 7003
   127.0.0.1:7003> get a
   (nil)
   127.0.0.1:7003> exit
   
   C:\Users\mao>
   ```

   



## 主从数据同步原理

### 全量同步

主从第一次建立连接时，会执行**全量同步**，将master节点的所有数据都拷贝给slave节点

第一阶段：

* slave：执行 replicaof命令， 建立连接
* slave：请求数据同步
* master：判断是否是第一次同步
* master：是第一次，返回master的数据版本信息
* slave：保存 版本信息

第二阶段：

* master：执行bgsave，生成RDB，并记录 RDB期间的所有命令到repl_baklog
* master：发送RDB文件
* slave：清空本地数据，加载RDB文件

第三阶段：

* matser：发送repl_baklog中的命令
* slave：执行接收到的命令



- **Replication Id**：简称replid，是数据集的标记，id一致则说明是同一数据集。每一个master都有唯一的replid，slave则会继承master节点的replid
- **offset**：偏移量，随着记录在repl_baklog中的数据增多而逐渐增大。slave完成同步时也会记录当前同步的offset。如果slave的offset小于master的offset，说明slave数据落后于master，需要更新。



master如何判断slave是不是第一次来同步数据？

lave做数据同步，必须向master声明自己的replication id 和offset，master才可以判断到底需要同步哪些数据。

因为slave原本也是一个master，有自己的replid和offset，当第一次变成slave，与master建立连接时，发送的replid和offset是自己的replid和offset。

master判断发现slave发送来的replid与自己的不一致，说明这是一个全新的slave，就知道要做全量同步了。

master会将自己的replid和offset都发送给这个slave，slave保存这些信息。以后slave的replid就与master一致了。

因此，**master判断一个节点是否是第一次同步的依据，就是看replid是否一致**。



完整流程描述：

- slave节点请求增量同步
- master节点判断replid，发现不一致，拒绝增量同步
- master将完整内存数据生成RDB，发送RDB到slave
- slave清空本地数据，加载master的RDB
- master将RDB期间的命令记录在repl_baklog，并持续将log中的命令发送给slave
- slave执行接收到的命令，保持与master之间的同步



###  增量同步

全量同步需要先做RDB，然后将RDB文件通过网络传输个slave，成本太高了。因此除了第一次做全量同步，其它大多数时候slave与master都是做**增量同步**。

什么是增量同步？就是只更新slave与master存在差异的部分数据。主从第一次同步是全量同步，但如果slave重启后同步，则执行增量同步

第一阶段：

* slave：重启
* slave：psync replid offset
* master：判断请求replid是否一致
* master：一致，不是第一次，返回continue

第二阶段：

* master：去repl_baklog 中获取offset后的数据
* master：发送offset后的命令
* slave：执行 命令





### repl_backlog原理

repl_baklog文件是一个固定大小的数组，只不过数组是环形，也就是说**角标到达数组末尾后，会再次从0开始读写**，这样数组头部的数据就会被覆盖。

repl_baklog中会记录Redis处理过的命令日志及offset，包括master当前的offset，和slave已经拷贝到的offset。

slave与master的offset之间的差异，就是salve需要增量拷贝的数据了。

随着不断有数据写入，master的offset逐渐变大，slave也不断的拷贝，追赶master的offset。

直到数组被填满。

此时，如果有新的数据写入，就会覆盖数组中的旧数据。

但是，如果slave出现网络阻塞，导致master的offset远远超过了slave的offset，如果master继续写入新数据，其offset就会覆盖旧的数据，直到将slave现在的offset也覆盖。此时如果slave恢复，需要同步，却发现自己的offset都没有了，无法完成增量同步了。只能做全量同步。



注意：repl_baklog大小有上限，写满后会覆盖最早的数据。如果slave断开时间过久，导 致尚未备份的数据被覆盖，则无法基于log做增量同步，只能再次全量同步。



## 主从同步优化

主从同步可以保证主从数据的一致性，非常重要。

可以从以下几个方面来优化Redis主从就集群：

- 在master中配置repl-diskless-sync yes启用无磁盘复制，避免全量同步时的磁盘IO。
- Redis单节点上的内存占用不要太大，减少RDB导致的过多磁盘IO
- 适当提高repl_baklog的大小，发现slave宕机时尽快实现故障恢复，尽可能避免全量同步
- 限制一个master上的slave节点数量，如果实在是太多slave，则可以采用主-从-从链式结构，减少master压力



## 全量同步和增量同步的区别

- 全量同步：master将完整内存数据生成RDB，发送RDB到slave。后续命令则记录在repl_baklog，逐个发送给slave。
- 增量同步：slave提交自己的offset到master，master获取repl_baklog中从offset之后的命令给slave

## 什么时候执行全量同步

- slave节点第一次连接master节点时
- slave节点断开时间太久，repl_baklog中的offset已经被覆盖时

## 什么时候执行增量同步

* slave节点断开又恢复，并且在repl_baklog中能找到offset时



## windows主从启动脚本

创建bat文件，输入以下命令：

```sh
start "redis-7001" redis-server.exe redis1/redis.conf
start "redis-7002" redis-server.exe redis2/redis.conf
start "redis-7003" redis-server.exe redis3/redis.conf
```





# redis哨兵

## 作用

- **监控**：Sentinel 会不断检查您的master和slave是否按预期工作
- **自动故障恢复**：如果master故障，Sentinel会将一个slave提升为master。当故障实例恢复后也以新的master为主
- **通知**：Sentinel充当Redis客户端的服务发现来源，当集群发生故障转移时，会将最新信息推送给Redis的客户端



## 集群监控原理

Sentinel基于心跳机制监测服务状态，每隔1秒向集群的每个实例发送ping命令：

* 主观下线：如果某sentinel节点发现某实例未在规定时间响应，则认为该实例**主观下线**。

* 客观下线：若超过指定数量（quorum）的sentinel都认为该实例主观下线，则该实例**客观下线**。quorum值最好超过Sentinel实例数量的一半。



## 集群故障恢复原理

一旦发现master故障，sentinel需要在salve中选择一个作为新的master，选择依据是这样的：

- 首先会判断slave节点与master节点断开时间长短，如果超过指定值（down-after-milliseconds * 10）则会排除该slave节点
- 然后判断slave节点的slave-priority值，越小优先级越高，如果是0则永不参与选举
- 如果slave-prority一样，则判断slave节点的offset值，越大说明数据越新，优先级越高
- 最后是判断slave节点的运行id大小，越小优先级越高。



## 如何实现切换

- sentinel给备选的slave1节点发送slaveof no one命令，让该节点成为master
- sentinel给所有其它slave发送slaveof 127.0.0.1 7002 命令，让这些slave成为新master的从节点，开始从新的master上同步数据。
- 最后，sentinel将故障节点标记为slave，当故障节点恢复后会自动成为新的master的slave节点



##  集群步骤

三个sentinel节点的信息如下：

* sentinel1：文件夹：./sentinel1   端口号：7501

* sentinel2：文件夹：./sentinel2   端口号：7502

* sentinel3：文件夹：./sentinel3   端口号：7503

一个master节点：

* master：文件夹：./redis1  端口号：7001

两个slave节点：

* slave1：文件夹：./redis2  端口号：7002
* slave2：文件夹：./redis3  端口号：7003



### 1.创建对应的文件夹

7001、7002和7003节点参考主从集群

### 2.分别在目录创建一个sentinel.conf文件

sentinel1：

```sh
port 7501
sentinel announce-ip 127.0.0.1
sentinel monitor mymaster 127.0.0.1 7001 2
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 60000
dir ./sentinel1
```

- `port 27001`：是当前sentinel实例的端口
- `sentinel monitor mymaster 192.168.150.101 7001 2`：指定主节点信息
  - `mymaster`：主节点名称，自定义，任意写
  - `192.168.150.101 7001`：主节点的ip和端口
  - `2`：选举master时的quorum值



sentinel2：

```sh
port 7502
sentinel announce-ip 127.0.0.1
sentinel monitor mymaster 127.0.0.1 7001 2
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 60000
dir ./sentinel2
```



sentinel3：

```sh
port 7503
sentinel announce-ip 127.0.0.1
sentinel monitor mymaster 127.0.0.1 7001 2
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 60000
dir ./sentinel3
```



### 3.启动

启动7001节点：

```sh

C:\Program Files\redis>redis-server.exe redis1/redis.conf
[5656] 22 May 21:56:08.584 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
[5656] 22 May 21:56:08.584 # Redis version=5.0.14.1, bits=64, commit=ec77f72d, modified=0, pid=5656, just started
[5656] 22 May 21:56:08.584 # Configuration loaded
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 5.0.14.1 (ec77f72d/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 7001
 |    `-._   `._    /     _.-'    |     PID: 5656
  `-._    `-._  `-./  _.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

[5656] 22 May 21:56:08.587 # Server initialized
[5656] 22 May 21:56:08.587 * DB loaded from disk: 0.000 seconds
[5656] 22 May 21:56:08.587 * Ready to accept connections
[5656] 22 May 21:56:10.999 * Replica 127.0.0.1:7002 asks for synchronization
[5656] 22 May 21:56:10.999 * Partial resynchronization not accepted: Replication ID mismatch (Replica asked for '27683014f9088f12c2dde31f7a054b287c953c3a', my replication IDs are 'a7f60c53ee08d8e73c81b2ddb5ecb604727511dc' and '0000000000000000000000000000000000000000')
[5656] 22 May 21:56:11.000 * Starting BGSAVE for SYNC with target: disk
[5656] 22 May 21:56:11.027 * Background saving started by pid 6956
[5656] 22 May 21:56:11.153 # fork operation complete
[5656] 22 May 21:56:11.165 * Background saving terminated with success
[5656] 22 May 21:56:11.171 * Synchronization with replica 127.0.0.1:7002 succeeded
[5656] 22 May 21:56:16.174 * Replica 127.0.0.1:7003 asks for synchronization
[5656] 22 May 21:56:16.174 * Partial resynchronization not accepted: Replication ID mismatch (Replica asked for 'ca9b9d1126598b78d418fb7bf136e91dd3d99174', my replication IDs are 'fb437967af3d1b63e3be97fe62b4853e954a20f1' and '0000000000000000000000000000000000000000')
[5656] 22 May 21:56:16.174 * Starting BGSAVE for SYNC with target: disk
[5656] 22 May 21:56:16.180 * Background saving started by pid 12796
[5656] 22 May 21:56:16.274 # fork operation complete
[5656] 22 May 21:56:16.284 * Background saving terminated with success
[5656] 22 May 21:56:16.289 * Synchronization with replica 127.0.0.1:7003 succeeded

```

启动7002节点

```sh

C:\Program Files\redis>redis-server.exe redis2/redis.conf
[3136] 22 May 21:56:10.994 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
[3136] 22 May 21:56:10.994 # Redis version=5.0.14.1, bits=64, commit=ec77f72d, modified=0, pid=3136, just started
[3136] 22 May 21:56:10.994 # Configuration loaded
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 5.0.14.1 (ec77f72d/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 7002
 |    `-._   `._    /     _.-'    |     PID: 3136
  `-._    `-._  `-./  _.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

[3136] 22 May 21:56:10.997 # Server initialized
[3136] 22 May 21:56:10.997 * DB loaded from disk: 0.000 seconds
[3136] 22 May 21:56:10.997 * Before turning into a replica, using my master parameters to synthesize a cached master: I may be able to synchronize with the new master with just a partial transfer.
[3136] 22 May 21:56:10.997 * Ready to accept connections
[3136] 22 May 21:56:10.997 * Connecting to MASTER 127.0.0.1:7001
[3136] 22 May 21:56:10.998 * MASTER <-> REPLICA sync started
[3136] 22 May 21:56:10.998 * Non blocking connect for SYNC fired the event.
[3136] 22 May 21:56:10.999 * Master replied to PING, replication can continue...
[3136] 22 May 21:56:10.999 * Trying a partial resynchronization (request 27683014f9088f12c2dde31f7a054b287c953c3a:1).
[3136] 22 May 21:56:11.027 * Full resync from master: fb437967af3d1b63e3be97fe62b4853e954a20f1:0
[3136] 22 May 21:56:11.028 * Discarding previously cached master state.
[3136] 22 May 21:56:11.171 * MASTER <-> REPLICA sync: receiving 189 bytes from master
[3136] 22 May 21:56:11.173 * MASTER <-> REPLICA sync: Flushing old data
[3136] 22 May 21:56:11.173 * MASTER <-> REPLICA sync: Loading DB in memory
[3136] 22 May 21:56:11.176 * MASTER <-> REPLICA sync: Finished with success

```



启动7003节点：

```sh

C:\Program Files\redis>redis-server.exe redis3/redis.conf
[17744] 22 May 21:56:16.168 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
[17744] 22 May 21:56:16.168 # Redis version=5.0.14.1, bits=64, commit=ec77f72d, modified=0, pid=17744, just started
[17744] 22 May 21:56:16.168 # Configuration loaded
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 5.0.14.1 (ec77f72d/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 7003
 |    `-._   `._    /     _.-'    |     PID: 17744
  `-._    `-._  `-./  _.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

[17744] 22 May 21:56:16.171 # Server initialized
[17744] 22 May 21:56:16.171 * DB loaded from disk: 0.000 seconds
[17744] 22 May 21:56:16.171 * Before turning into a replica, using my master parameters to synthesize a cached master: I may be able to synchronize with the new master with just a partial transfer.
[17744] 22 May 21:56:16.172 * Ready to accept connections
[17744] 22 May 21:56:16.172 * Connecting to MASTER 127.0.0.1:7001
[17744] 22 May 21:56:16.172 * MASTER <-> REPLICA sync started
[17744] 22 May 21:56:16.172 * Non blocking connect for SYNC fired the event.
[17744] 22 May 21:56:16.173 * Master replied to PING, replication can continue...
[17744] 22 May 21:56:16.173 * Trying a partial resynchronization (request ca9b9d1126598b78d418fb7bf136e91dd3d99174:1435).
[17744] 22 May 21:56:16.181 * Full resync from master: fb437967af3d1b63e3be97fe62b4853e954a20f1:0
[17744] 22 May 21:56:16.181 * Discarding previously cached master state.
[17744] 22 May 21:56:16.289 * MASTER <-> REPLICA sync: receiving 189 bytes from master
[17744] 22 May 21:56:16.290 * MASTER <-> REPLICA sync: Flushing old data
[17744] 22 May 21:56:16.293 * MASTER <-> REPLICA sync: Loading DB in memory
[17744] 22 May 21:56:16.294 * MASTER <-> REPLICA sync: Finished with success

```





启动三个哨兵服务：

```sh
# 第1个
redis-server.exe sentinel1/sentinel.conf --sentinel
# 第2个
redis-server.exe sentinel2/sentinel.conf --sentinel
# 第3个
redis-server.exe sentinel3/sentinel.conf --sentinel
```



第一个：

```sh

C:\Program Files\redis>redis-server.exe sentinel1/sentinel.conf --sentinel
[15520] 22 May 22:00:03.413 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
[15520] 22 May 22:00:03.413 # Redis version=5.0.14.1, bits=64, commit=ec77f72d, modified=0, pid=15520, just started
[15520] 22 May 22:00:03.413 # Configuration loaded
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 5.0.14.1 (ec77f72d/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in sentinel mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 7501
 |    `-._   `._    /     _.-'    |     PID: 15520
  `-._    `-._  `-./  _.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

[15520] 22 May 22:00:03.416 # Sentinel ID is 998f885399f6bcf3da404d8d332653d6c4137eff
[15520] 22 May 22:00:03.416 # +monitor master mymaster 127.0.0.1 7001 quorum 2
[15520] 22 May 22:00:03.418 * +slave slave 127.0.0.1:7002 127.0.0.1 7002 @ mymaster 127.0.0.1 7001
[15520] 22 May 22:00:03.425 * +slave slave 127.0.0.1:7003 127.0.0.1 7003 @ mymaster 127.0.0.1 7001

```

第二个：

```sh

C:\Program Files\redis>redis-server.exe sentinel2/sentinel.conf --sentinel
[11912] 22 May 22:01:23.955 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
[11912] 22 May 22:01:23.955 # Redis version=5.0.14.1, bits=64, commit=ec77f72d, modified=0, pid=11912, just started
[11912] 22 May 22:01:23.955 # Configuration loaded
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 5.0.14.1 (ec77f72d/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in sentinel mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 7502
 |    `-._   `._    /     _.-'    |     PID: 11912
  `-._    `-._  `-./  _.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

[11912] 22 May 22:01:23.958 # Sentinel ID is 0fa1387c46f90018527a3fa15771de5091da855a
[11912] 22 May 22:01:23.958 # +monitor master mymaster 127.0.0.1 7001 quorum 2
[11912] 22 May 22:01:23.968 * +slave slave 127.0.0.1:7002 127.0.0.1 7002 @ mymaster 127.0.0.1 7001
[11912] 22 May 22:01:23.973 * +slave slave 127.0.0.1:7003 127.0.0.1 7003 @ mymaster 127.0.0.1 7001
[11912] 22 May 22:01:24.904 * +sentinel sentinel 998f885399f6bcf3da404d8d332653d6c4137eff 127.0.0.1 7501 @ mymaster 127.0.0.1 7001

```

第三个：

```sh

C:\Program Files\redis>redis-server.exe sentinel3/sentinel.conf --sentinel
[18740] 22 May 22:02:19.809 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
[18740] 22 May 22:02:19.810 # Redis version=5.0.14.1, bits=64, commit=ec77f72d, modified=0, pid=18740, just started
[18740] 22 May 22:02:19.810 # Configuration loaded
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 5.0.14.1 (ec77f72d/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in sentinel mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 7503
 |    `-._   `._    /     _.-'    |     PID: 18740
  `-._    `-._  `-./  _.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

[18740] 22 May 22:02:19.818 # Sentinel ID is b7fd68ca34e48df8850e596a35935a67a324aeae
[18740] 22 May 22:02:19.818 # +monitor master mymaster 127.0.0.1 7001 quorum 2
[18740] 22 May 22:02:19.819 * +slave slave 127.0.0.1:7002 127.0.0.1 7002 @ mymaster 127.0.0.1 7001
[18740] 22 May 22:02:19.825 * +slave slave 127.0.0.1:7003 127.0.0.1 7003 @ mymaster 127.0.0.1 7001
[18740] 22 May 22:02:20.026 * +sentinel sentinel 998f885399f6bcf3da404d8d332653d6c4137eff 127.0.0.1 7501 @ mymaster 127.0.0.1 7001
[18740] 22 May 22:02:21.248 * +sentinel sentinel 0fa1387c46f90018527a3fa15771de5091da855a 127.0.0.1 7502 @ mymaster 127.0.0.1 7001

```



### 4. 测试关闭master

关闭master节点(7001节点)：

```sh

C:\Program Files\redis>redis-server.exe redis1/redis.conf
[5656] 22 May 21:56:08.584 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
[5656] 22 May 21:56:08.584 # Redis version=5.0.14.1, bits=64, commit=ec77f72d, modified=0, pid=5656, just started
[5656] 22 May 21:56:08.584 # Configuration loaded
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 5.0.14.1 (ec77f72d/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 7001
 |    `-._   `._    /     _.-'    |     PID: 5656
  `-._    `-._  `-./  _.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

[5656] 22 May 21:56:08.587 # Server initialized
[5656] 22 May 21:56:08.587 * DB loaded from disk: 0.000 seconds
[5656] 22 May 21:56:08.587 * Ready to accept connections
[5656] 22 May 21:56:10.999 * Replica 127.0.0.1:7002 asks for synchronization
[5656] 22 May 21:56:10.999 * Partial resynchronization not accepted: Replication ID mismatch (Replica asked for '27683014f9088f12c2dde31f7a054b287c953c3a', my replication IDs are 'a7f60c53ee08d8e73c81b2ddb5ecb604727511dc' and '0000000000000000000000000000000000000000')
[5656] 22 May 21:56:11.000 * Starting BGSAVE for SYNC with target: disk
[5656] 22 May 21:56:11.027 * Background saving started by pid 6956
[5656] 22 May 21:56:11.153 # fork operation complete
[5656] 22 May 21:56:11.165 * Background saving terminated with success
[5656] 22 May 21:56:11.171 * Synchronization with replica 127.0.0.1:7002 succeeded
[5656] 22 May 21:56:16.174 * Replica 127.0.0.1:7003 asks for synchronization
[5656] 22 May 21:56:16.174 * Partial resynchronization not accepted: Replication ID mismatch (Replica asked for 'ca9b9d1126598b78d418fb7bf136e91dd3d99174', my replication IDs are 'fb437967af3d1b63e3be97fe62b4853e954a20f1' and '0000000000000000000000000000000000000000')
[5656] 22 May 21:56:16.174 * Starting BGSAVE for SYNC with target: disk
[5656] 22 May 21:56:16.180 * Background saving started by pid 12796
[5656] 22 May 21:56:16.274 # fork operation complete
[5656] 22 May 21:56:16.284 * Background saving terminated with success
[5656] 22 May 21:56:16.289 * Synchronization with replica 127.0.0.1:7003 succeeded
[5656] 22 May 22:07:18.096 # User requested shutdown...
[5656] 22 May 22:07:18.096 * Saving the final RDB snapshot before exiting.
[5656] 22 May 22:07:18.098 * DB saved on disk
[5656] 22 May 22:07:18.098 # Redis is now ready to exit, bye bye...
终止批处理操作吗(Y/N)?
```



7002节点：

```sh

C:\Program Files\redis>redis-server.exe redis2/redis.conf
[3136] 22 May 21:56:10.994 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
[3136] 22 May 21:56:10.994 # Redis version=5.0.14.1, bits=64, commit=ec77f72d, modified=0, pid=3136, just started
[3136] 22 May 21:56:10.994 # Configuration loaded
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 5.0.14.1 (ec77f72d/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 7002
 |    `-._   `._    /     _.-'    |     PID: 3136
  `-._    `-._  `-./  _.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

[3136] 22 May 21:56:10.997 # Server initialized
[3136] 22 May 21:56:10.997 * DB loaded from disk: 0.000 seconds
[3136] 22 May 21:56:10.997 * Before turning into a replica, using my master parameters to synthesize a cached master: I may be able to synchronize with the new master with just a partial transfer.
[3136] 22 May 21:56:10.997 * Ready to accept connections
[3136] 22 May 21:56:10.997 * Connecting to MASTER 127.0.0.1:7001
[3136] 22 May 21:56:10.998 * MASTER <-> REPLICA sync started
[3136] 22 May 21:56:10.998 * Non blocking connect for SYNC fired the event.
[3136] 22 May 21:56:10.999 * Master replied to PING, replication can continue...
[3136] 22 May 21:56:10.999 * Trying a partial resynchronization (request 27683014f9088f12c2dde31f7a054b287c953c3a:1).
[3136] 22 May 21:56:11.027 * Full resync from master: fb437967af3d1b63e3be97fe62b4853e954a20f1:0
[3136] 22 May 21:56:11.028 * Discarding previously cached master state.
[3136] 22 May 21:56:11.171 * MASTER <-> REPLICA sync: receiving 189 bytes from master
[3136] 22 May 21:56:11.173 * MASTER <-> REPLICA sync: Flushing old data
[3136] 22 May 21:56:11.173 * MASTER <-> REPLICA sync: Loading DB in memory
[3136] 22 May 21:56:11.176 * MASTER <-> REPLICA sync: Finished with success
[3136] 22 May 22:07:18.099 # Connection with master lost.
[3136] 22 May 22:07:18.099 * Caching the disconnected master state.
[3136] 22 May 22:07:18.792 * Connecting to MASTER 127.0.0.1:7001
[3136] 22 May 22:07:18.792 * MASTER <-> REPLICA sync started
[3136] 22 May 22:07:20.822 * Non blocking connect for SYNC fired the event.
[3136] 22 May 22:07:20.822 # Sending command to master in replication handshake: -Writing to master: 找不到指 定的模块。
[3136] 22 May 22:07:21.011 * Connecting to MASTER 127.0.0.1:7001
[3136] 22 May 22:07:21.011 * MASTER <-> REPLICA sync started
[3136] 22 May 22:07:23.043 * Non blocking connect for SYNC fired the event.
[3136] 22 May 22:07:23.043 # Sending command to master in replication handshake: -Writing to master: 找不到指 定的模块。
[3136] 22 May 22:07:23.233 * Connecting to MASTER 127.0.0.1:7001
[3136] 22 May 22:07:23.233 * MASTER <-> REPLICA sync started
[3136] 22 May 22:07:23.439 # Setting secondary replication ID to fb437967af3d1b63e3be97fe62b4853e954a20f1, valid up to offset: 70956. New replication ID is 710dc6e5b7f14e9a40fdf25833a7809e5ac18791
[3136] 22 May 22:07:23.439 * Discarding previously cached master state.
[3136] 22 May 22:07:23.440 * MASTER MODE enabled (user request from 'id=5 addr=127.0.0.1:65454 fd=9 name=sentinel-998f8853-cmd age=440 idle=0 flags=x db=0 sub=0 psub=0 multi=3 qbuf=140 qbuf-free=32628 obl=36 oll=0 omem=0 events=r cmd=exec')
[3136] 22 May 22:07:23.452 # CONFIG REWRITE executed with success.
[3136] 22 May 22:07:25.077 * Replica 127.0.0.1:7003 asks for synchronization
[3136] 22 May 22:07:25.077 * Partial resynchronization request from 127.0.0.1:7003 accepted. Sending 551 bytes of backlog starting from offset 70956.

```



7003节点：

```sh

C:\Program Files\redis>redis-server.exe redis3/redis.conf
[17744] 22 May 21:56:16.168 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
[17744] 22 May 21:56:16.168 # Redis version=5.0.14.1, bits=64, commit=ec77f72d, modified=0, pid=17744, just started
[17744] 22 May 21:56:16.168 # Configuration loaded
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 5.0.14.1 (ec77f72d/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 7003
 |    `-._   `._    /     _.-'    |     PID: 17744
  `-._    `-._  `-./  _.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

[17744] 22 May 21:56:16.171 # Server initialized
[17744] 22 May 21:56:16.171 * DB loaded from disk: 0.000 seconds
[17744] 22 May 21:56:16.171 * Before turning into a replica, using my master parameters to synthesize a cached master: I may be able to synchronize with the new master with just a partial transfer.
[17744] 22 May 21:56:16.172 * Ready to accept connections
[17744] 22 May 21:56:16.172 * Connecting to MASTER 127.0.0.1:7001
[17744] 22 May 21:56:16.172 * MASTER <-> REPLICA sync started
[17744] 22 May 21:56:16.172 * Non blocking connect for SYNC fired the event.
[17744] 22 May 21:56:16.173 * Master replied to PING, replication can continue...
[17744] 22 May 21:56:16.173 * Trying a partial resynchronization (request ca9b9d1126598b78d418fb7bf136e91dd3d99174:1435).
[17744] 22 May 21:56:16.181 * Full resync from master: fb437967af3d1b63e3be97fe62b4853e954a20f1:0
[17744] 22 May 21:56:16.181 * Discarding previously cached master state.
[17744] 22 May 21:56:16.289 * MASTER <-> REPLICA sync: receiving 189 bytes from master
[17744] 22 May 21:56:16.290 * MASTER <-> REPLICA sync: Flushing old data
[17744] 22 May 21:56:16.293 * MASTER <-> REPLICA sync: Loading DB in memory
[17744] 22 May 21:56:16.294 * MASTER <-> REPLICA sync: Finished with success
[17744] 22 May 22:07:18.098 # Connection with master lost.
[17744] 22 May 22:07:18.099 * Caching the disconnected master state.
[17744] 22 May 22:07:18.412 * Connecting to MASTER 127.0.0.1:7001
[17744] 22 May 22:07:18.412 * MASTER <-> REPLICA sync started
[17744] 22 May 22:07:20.441 * Non blocking connect for SYNC fired the event.
[17744] 22 May 22:07:20.441 # Sending command to master in replication handshake: -Writing to master: 找不到指定的模块。
[17744] 22 May 22:07:20.632 * Connecting to MASTER 127.0.0.1:7001
[17744] 22 May 22:07:20.632 * MASTER <-> REPLICA sync started
[17744] 22 May 22:07:22.661 * Non blocking connect for SYNC fired the event.
[17744] 22 May 22:07:22.661 # Sending command to master in replication handshake: -Writing to master: 找不到指定的模块。
[17744] 22 May 22:07:22.852 * Connecting to MASTER 127.0.0.1:7001
[17744] 22 May 22:07:22.852 * MASTER <-> REPLICA sync started
[17744] 22 May 22:07:24.311 * REPLICAOF 127.0.0.1:7002 enabled (user request from 'id=5 addr=127.0.0.1:65456 fd=9 name=sentinel-998f8853-cmd age=441 idle=0 flags=x db=0 sub=0 psub=0 multi=3 qbuf=280 qbuf-free=32488 obl=36 oll=0 omem=0 events=r cmd=exec')
[17744] 22 May 22:07:24.324 # CONFIG REWRITE executed with success.
[17744] 22 May 22:07:25.075 * Connecting to MASTER 127.0.0.1:7002
[17744] 22 May 22:07:25.075 * MASTER <-> REPLICA sync started
[17744] 22 May 22:07:25.076 * Non blocking connect for SYNC fired the event.
[17744] 22 May 22:07:25.076 * Master replied to PING, replication can continue...
[17744] 22 May 22:07:25.077 * Trying a partial resynchronization (request fb437967af3d1b63e3be97fe62b4853e954a20f1:70956).
[17744] 22 May 22:07:25.077 * Successful partial resynchronization with master.
[17744] 22 May 22:07:25.078 # Master replication ID changed to 710dc6e5b7f14e9a40fdf25833a7809e5ac18791
[17744] 22 May 22:07:25.078 * MASTER <-> REPLICA sync: Master accepted a Partial Resynchronization.

```



哨兵7501节点：

```sh

C:\Program Files\redis>redis-server.exe sentinel1/sentinel.conf --sentinel
[15520] 22 May 22:00:03.413 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
[15520] 22 May 22:00:03.413 # Redis version=5.0.14.1, bits=64, commit=ec77f72d, modified=0, pid=15520, just started
[15520] 22 May 22:00:03.413 # Configuration loaded
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 5.0.14.1 (ec77f72d/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in sentinel mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 7501
 |    `-._   `._    /     _.-'    |     PID: 15520
  `-._    `-._  `-./  _.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

[15520] 22 May 22:00:03.416 # Sentinel ID is 998f885399f6bcf3da404d8d332653d6c4137eff
[15520] 22 May 22:00:03.416 # +monitor master mymaster 127.0.0.1 7001 quorum 2
[15520] 22 May 22:00:03.418 * +slave slave 127.0.0.1:7002 127.0.0.1 7002 @ mymaster 127.0.0.1 7001
[15520] 22 May 22:00:03.425 * +slave slave 127.0.0.1:7003 127.0.0.1 7003 @ mymaster 127.0.0.1 7001
[15520] 22 May 22:01:25.957 * +sentinel sentinel 0fa1387c46f90018527a3fa15771de5091da855a 127.0.0.1 7502 @ mymaster 127.0.0.1 7001
[15520] 22 May 22:02:21.870 * +sentinel sentinel b7fd68ca34e48df8850e596a35935a67a324aeae 127.0.0.1 7503 @ mymaster 127.0.0.1 7001
[15520] 22 May 22:07:23.170 # +sdown master mymaster 127.0.0.1 7001
[15520] 22 May 22:07:23.233 # +odown master mymaster 127.0.0.1 7001 #quorum 2/2
[15520] 22 May 22:07:23.233 # +new-epoch 1
[15520] 22 May 22:07:23.233 # +try-failover master mymaster 127.0.0.1 7001
[15520] 22 May 22:07:23.241 # +vote-for-leader 998f885399f6bcf3da404d8d332653d6c4137eff 1
[15520] 22 May 22:07:23.252 # b7fd68ca34e48df8850e596a35935a67a324aeae voted for 998f885399f6bcf3da404d8d332653d6c4137eff 1
[15520] 22 May 22:07:23.254 # 0fa1387c46f90018527a3fa15771de5091da855a voted for 998f885399f6bcf3da404d8d332653d6c4137eff 1
[15520] 22 May 22:07:23.297 # +elected-leader master mymaster 127.0.0.1 7001
[15520] 22 May 22:07:23.297 # +failover-state-select-slave master mymaster 127.0.0.1 7001
[15520] 22 May 22:07:23.361 # +selected-slave slave 127.0.0.1:7002 127.0.0.1 7002 @ mymaster 127.0.0.1 7001
[15520] 22 May 22:07:23.361 * +failover-state-send-slaveof-noone slave 127.0.0.1:7002 127.0.0.1 7002 @ mymaster 127.0.0.1 7001
[15520] 22 May 22:07:23.439 * +failover-state-wait-promotion slave 127.0.0.1:7002 127.0.0.1 7002 @ mymaster 127.0.0.1 7001
[15520] 22 May 22:07:24.254 # +promoted-slave slave 127.0.0.1:7002 127.0.0.1 7002 @ mymaster 127.0.0.1 7001
[15520] 22 May 22:07:24.254 # +failover-state-reconf-slaves master mymaster 127.0.0.1 7001
[15520] 22 May 22:07:24.311 * +slave-reconf-sent slave 127.0.0.1:7003 127.0.0.1 7003 @ mymaster 127.0.0.1 7001
[15520] 22 May 22:07:25.282 * +slave-reconf-inprog slave 127.0.0.1:7003 127.0.0.1 7003 @ mymaster 127.0.0.1 7001
[15520] 22 May 22:07:25.282 * +slave-reconf-done slave 127.0.0.1:7003 127.0.0.1 7003 @ mymaster 127.0.0.1 7001
[15520] 22 May 22:07:25.362 # +failover-end master mymaster 127.0.0.1 7001
[15520] 22 May 22:07:25.362 # +switch-master mymaster 127.0.0.1 7001 127.0.0.1 7002
[15520] 22 May 22:07:25.369 * +slave slave 127.0.0.1:7003 127.0.0.1 7003 @ mymaster 127.0.0.1 7002
[15520] 22 May 22:07:25.374 * +slave slave 127.0.0.1:7001 127.0.0.1 7001 @ mymaster 127.0.0.1 7002
[15520] 22 May 22:07:30.438 # +sdown slave 127.0.0.1:7001 127.0.0.1 7001 @ mymaster 127.0.0.1 7002

```



哨兵7502节点：

```sh

C:\Program Files\redis>redis-server.exe sentinel2/sentinel.conf --sentinel
[11912] 22 May 22:01:23.955 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
[11912] 22 May 22:01:23.955 # Redis version=5.0.14.1, bits=64, commit=ec77f72d, modified=0, pid=11912, just started
[11912] 22 May 22:01:23.955 # Configuration loaded
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 5.0.14.1 (ec77f72d/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in sentinel mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 7502
 |    `-._   `._    /     _.-'    |     PID: 11912
  `-._    `-._  `-./  _.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

[11912] 22 May 22:01:23.958 # Sentinel ID is 0fa1387c46f90018527a3fa15771de5091da855a
[11912] 22 May 22:01:23.958 # +monitor master mymaster 127.0.0.1 7001 quorum 2
[11912] 22 May 22:01:23.968 * +slave slave 127.0.0.1:7002 127.0.0.1 7002 @ mymaster 127.0.0.1 7001
[11912] 22 May 22:01:23.973 * +slave slave 127.0.0.1:7003 127.0.0.1 7003 @ mymaster 127.0.0.1 7001
[11912] 22 May 22:01:24.904 * +sentinel sentinel 998f885399f6bcf3da404d8d332653d6c4137eff 127.0.0.1 7501 @ mymaster 127.0.0.1 7001
[11912] 22 May 22:02:21.870 * +sentinel sentinel b7fd68ca34e48df8850e596a35935a67a324aeae 127.0.0.1 7503 @ mymaster 127.0.0.1 7001
[11912] 22 May 22:07:23.233 # +sdown master mymaster 127.0.0.1 7001
[11912] 22 May 22:07:23.247 # +new-epoch 1
[11912] 22 May 22:07:23.254 # +vote-for-leader 998f885399f6bcf3da404d8d332653d6c4137eff 1
[11912] 22 May 22:07:23.345 # +odown master mymaster 127.0.0.1 7001 #quorum 3/2
[11912] 22 May 22:07:23.345 # Next failover delay: I will not start a failover before Sun May 22 22:09:23 2022
[11912] 22 May 22:07:24.311 # +config-update-from sentinel 998f885399f6bcf3da404d8d332653d6c4137eff 127.0.0.1 7501 @ mymaster 127.0.0.1 7001
[11912] 22 May 22:07:24.311 # +switch-master mymaster 127.0.0.1 7001 127.0.0.1 7002
[11912] 22 May 22:07:24.313 * +slave slave 127.0.0.1:7003 127.0.0.1 7003 @ mymaster 127.0.0.1 7002
[11912] 22 May 22:07:24.317 * +slave slave 127.0.0.1:7001 127.0.0.1 7001 @ mymaster 127.0.0.1 7002
[11912] 22 May 22:07:29.360 # +sdown slave 127.0.0.1:7001 127.0.0.1 7001 @ mymaster 127.0.0.1 7002

```



哨兵7503节点：

```sh

C:\Program Files\redis>redis-server.exe sentinel3/sentinel.conf --sentinel
[18740] 22 May 22:02:19.809 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
[18740] 22 May 22:02:19.810 # Redis version=5.0.14.1, bits=64, commit=ec77f72d, modified=0, pid=18740, just started
[18740] 22 May 22:02:19.810 # Configuration loaded
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 5.0.14.1 (ec77f72d/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in sentinel mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 7503
 |    `-._   `._    /     _.-'    |     PID: 18740
  `-._    `-._  `-./  _.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

[18740] 22 May 22:02:19.818 # Sentinel ID is b7fd68ca34e48df8850e596a35935a67a324aeae
[18740] 22 May 22:02:19.818 # +monitor master mymaster 127.0.0.1 7001 quorum 2
[18740] 22 May 22:02:19.819 * +slave slave 127.0.0.1:7002 127.0.0.1 7002 @ mymaster 127.0.0.1 7001
[18740] 22 May 22:02:19.825 * +slave slave 127.0.0.1:7003 127.0.0.1 7003 @ mymaster 127.0.0.1 7001
[18740] 22 May 22:02:20.026 * +sentinel sentinel 998f885399f6bcf3da404d8d332653d6c4137eff 127.0.0.1 7501 @ mymaster 127.0.0.1 7001
[18740] 22 May 22:02:21.248 * +sentinel sentinel 0fa1387c46f90018527a3fa15771de5091da855a 127.0.0.1 7502 @ mymaster 127.0.0.1 7001
[18740] 22 May 22:07:23.170 # +sdown master mymaster 127.0.0.1 7001
[18740] 22 May 22:07:23.246 # +new-epoch 1
[18740] 22 May 22:07:23.252 # +vote-for-leader 998f885399f6bcf3da404d8d332653d6c4137eff 1
[18740] 22 May 22:07:23.281 # +odown master mymaster 127.0.0.1 7001 #quorum 2/2
[18740] 22 May 22:07:23.281 # Next failover delay: I will not start a failover before Sun May 22 22:09:23 2022
[18740] 22 May 22:07:24.311 # +config-update-from sentinel 998f885399f6bcf3da404d8d332653d6c4137eff 127.0.0.1 7501 @ mymaster 127.0.0.1 7001
[18740] 22 May 22:07:24.312 # +switch-master mymaster 127.0.0.1 7001 127.0.0.1 7002
[18740] 22 May 22:07:24.313 * +slave slave 127.0.0.1:7003 127.0.0.1 7003 @ mymaster 127.0.0.1 7002
[18740] 22 May 22:07:24.317 * +slave slave 127.0.0.1:7001 127.0.0.1 7001 @ mymaster 127.0.0.1 7002
[18740] 22 May 22:07:29.328 # +sdown slave 127.0.0.1:7001 127.0.0.1 7001 @ mymaster 127.0.0.1 7002

```



### 5.测试重新启动master

重新启动7001节点

```sh

C:\Program Files\redis>redis-server.exe redis1/redis.conf
[16868] 22 May 22:15:10.604 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
[16868] 22 May 22:15:10.604 # Redis version=5.0.14.1, bits=64, commit=ec77f72d, modified=0, pid=16868, just started
[16868] 22 May 22:15:10.605 # Configuration loaded
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 5.0.14.1 (ec77f72d/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 7001
 |    `-._   `._    /     _.-'    |     PID: 16868
  `-._    `-._  `-./  _.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

[16868] 22 May 22:15:10.608 # Server initialized
[16868] 22 May 22:15:10.612 * DB loaded from disk: 0.004 seconds
[16868] 22 May 22:15:10.612 * Ready to accept connections
[16868] 22 May 22:15:22.995 * Before turning into a replica, using my master parameters to synthesize a cached master: I may be able to synchronize with the new master with just a partial transfer.
[16868] 22 May 22:15:22.995 * REPLICAOF 127.0.0.1:7002 enabled (user request from 'id=3 addr=127.0.0.1:49843 fd=9 name=sentinel-b7fd68ca-cmd age=10 idle=0 flags=x db=0 sub=0 psub=0 multi=3 qbuf=148 qbuf-free=32620 obl=36 oll=0 omem=0 events=r cmd=exec')
[16868] 22 May 22:15:22.999 # CONFIG REWRITE executed with success.
[16868] 22 May 22:15:23.982 * Connecting to MASTER 127.0.0.1:7002
[16868] 22 May 22:15:23.982 * MASTER <-> REPLICA sync started
[16868] 22 May 22:15:23.983 * Non blocking connect for SYNC fired the event.
[16868] 22 May 22:15:23.983 * Master replied to PING, replication can continue...
[16868] 22 May 22:15:23.983 * Trying a partial resynchronization (request 8b867f59f828ddf06fca6540e333b40c3834c0fe:1).
[16868] 22 May 22:15:23.995 * Full resync from master: 710dc6e5b7f14e9a40fdf25833a7809e5ac18791:164904
[16868] 22 May 22:15:23.995 * Discarding previously cached master state.
[16868] 22 May 22:15:24.181 * MASTER <-> REPLICA sync: receiving 192 bytes from master
[16868] 22 May 22:15:24.183 * MASTER <-> REPLICA sync: Flushing old data
[16868] 22 May 22:15:24.183 * MASTER <-> REPLICA sync: Loading DB in memory
[16868] 22 May 22:15:24.185 * MASTER <-> REPLICA sync: Finished with success

```



其它节点变化：

7002节点：

```sh

C:\Program Files\redis>redis-server.exe redis2/redis.conf
[3136] 22 May 21:56:10.994 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
[3136] 22 May 21:56:10.994 # Redis version=5.0.14.1, bits=64, commit=ec77f72d, modified=0, pid=3136, just started
[3136] 22 May 21:56:10.994 # Configuration loaded
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 5.0.14.1 (ec77f72d/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 7002
 |    `-._   `._    /     _.-'    |     PID: 3136
  `-._    `-._  `-./  _.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

[3136] 22 May 21:56:10.997 # Server initialized
[3136] 22 May 21:56:10.997 * DB loaded from disk: 0.000 seconds
[3136] 22 May 21:56:10.997 * Before turning into a replica, using my master parameters to synthesize a cached master: I may be able to synchronize with the new master with just a partial transfer.
[3136] 22 May 21:56:10.997 * Ready to accept connections
[3136] 22 May 21:56:10.997 * Connecting to MASTER 127.0.0.1:7001
[3136] 22 May 21:56:10.998 * MASTER <-> REPLICA sync started
[3136] 22 May 21:56:10.998 * Non blocking connect for SYNC fired the event.
[3136] 22 May 21:56:10.999 * Master replied to PING, replication can continue...
[3136] 22 May 21:56:10.999 * Trying a partial resynchronization (request 27683014f9088f12c2dde31f7a054b287c953c3a:1).
[3136] 22 May 21:56:11.027 * Full resync from master: fb437967af3d1b63e3be97fe62b4853e954a20f1:0
[3136] 22 May 21:56:11.028 * Discarding previously cached master state.
[3136] 22 May 21:56:11.171 * MASTER <-> REPLICA sync: receiving 189 bytes from master
[3136] 22 May 21:56:11.173 * MASTER <-> REPLICA sync: Flushing old data
[3136] 22 May 21:56:11.173 * MASTER <-> REPLICA sync: Loading DB in memory
[3136] 22 May 21:56:11.176 * MASTER <-> REPLICA sync: Finished with success
[3136] 22 May 22:07:18.099 # Connection with master lost.
[3136] 22 May 22:07:18.099 * Caching the disconnected master state.
[3136] 22 May 22:07:18.792 * Connecting to MASTER 127.0.0.1:7001
[3136] 22 May 22:07:18.792 * MASTER <-> REPLICA sync started
[3136] 22 May 22:07:20.822 * Non blocking connect for SYNC fired the event.
[3136] 22 May 22:07:20.822 # Sending command to master in replication handshake: -Writing to master: 找不到指 定的模块。
[3136] 22 May 22:07:21.011 * Connecting to MASTER 127.0.0.1:7001
[3136] 22 May 22:07:21.011 * MASTER <-> REPLICA sync started
[3136] 22 May 22:07:23.043 * Non blocking connect for SYNC fired the event.
[3136] 22 May 22:07:23.043 # Sending command to master in replication handshake: -Writing to master: 找不到指 定的模块。
[3136] 22 May 22:07:23.233 * Connecting to MASTER 127.0.0.1:7001
[3136] 22 May 22:07:23.233 * MASTER <-> REPLICA sync started
[3136] 22 May 22:07:23.439 # Setting secondary replication ID to fb437967af3d1b63e3be97fe62b4853e954a20f1, valid up to offset: 70956. New replication ID is 710dc6e5b7f14e9a40fdf25833a7809e5ac18791
[3136] 22 May 22:07:23.439 * Discarding previously cached master state.
[3136] 22 May 22:07:23.440 * MASTER MODE enabled (user request from 'id=5 addr=127.0.0.1:65454 fd=9 name=sentinel-998f8853-cmd age=440 idle=0 flags=x db=0 sub=0 psub=0 multi=3 qbuf=140 qbuf-free=32628 obl=36 oll=0 omem=0 events=r cmd=exec')
[3136] 22 May 22:07:23.452 # CONFIG REWRITE executed with success.
[3136] 22 May 22:07:25.077 * Replica 127.0.0.1:7003 asks for synchronization
[3136] 22 May 22:07:25.077 * Partial resynchronization request from 127.0.0.1:7003 accepted. Sending 551 bytes of backlog starting from offset 70956.
[3136] 22 May 22:15:23.983 * Replica 127.0.0.1:7001 asks for synchronization
[3136] 22 May 22:15:23.984 * Partial resynchronization not accepted: Replication ID mismatch (Replica asked for '8b867f59f828ddf06fca6540e333b40c3834c0fe', my replication IDs are '710dc6e5b7f14e9a40fdf25833a7809e5ac18791' and 'fb437967af3d1b63e3be97fe62b4853e954a20f1')
[3136] 22 May 22:15:23.987 * Starting BGSAVE for SYNC with target: disk
[3136] 22 May 22:15:23.995 * Background saving started by pid 8712
[3136] 22 May 22:15:24.157 # fork operation complete
[3136] 22 May 22:15:24.175 * Background saving terminated with success
[3136] 22 May 22:15:24.182 * Synchronization with replica 127.0.0.1:7001 succeeded

```



7003节点：

```sh

C:\Program Files\redis>redis-server.exe redis3/redis.conf
[17744] 22 May 21:56:16.168 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
[17744] 22 May 21:56:16.168 # Redis version=5.0.14.1, bits=64, commit=ec77f72d, modified=0, pid=17744, just started
[17744] 22 May 21:56:16.168 # Configuration loaded
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 5.0.14.1 (ec77f72d/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 7003
 |    `-._   `._    /     _.-'    |     PID: 17744
  `-._    `-._  `-./  _.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

[17744] 22 May 21:56:16.171 # Server initialized
[17744] 22 May 21:56:16.171 * DB loaded from disk: 0.000 seconds
[17744] 22 May 21:56:16.171 * Before turning into a replica, using my master parameters to synthesize a cached master: I may be able to synchronize with the new master with just a partial transfer.
[17744] 22 May 21:56:16.172 * Ready to accept connections
[17744] 22 May 21:56:16.172 * Connecting to MASTER 127.0.0.1:7001
[17744] 22 May 21:56:16.172 * MASTER <-> REPLICA sync started
[17744] 22 May 21:56:16.172 * Non blocking connect for SYNC fired the event.
[17744] 22 May 21:56:16.173 * Master replied to PING, replication can continue...
[17744] 22 May 21:56:16.173 * Trying a partial resynchronization (request ca9b9d1126598b78d418fb7bf136e91dd3d99174:1435).
[17744] 22 May 21:56:16.181 * Full resync from master: fb437967af3d1b63e3be97fe62b4853e954a20f1:0
[17744] 22 May 21:56:16.181 * Discarding previously cached master state.
[17744] 22 May 21:56:16.289 * MASTER <-> REPLICA sync: receiving 189 bytes from master
[17744] 22 May 21:56:16.290 * MASTER <-> REPLICA sync: Flushing old data
[17744] 22 May 21:56:16.293 * MASTER <-> REPLICA sync: Loading DB in memory
[17744] 22 May 21:56:16.294 * MASTER <-> REPLICA sync: Finished with success
[17744] 22 May 22:07:18.098 # Connection with master lost.
[17744] 22 May 22:07:18.099 * Caching the disconnected master state.
[17744] 22 May 22:07:18.412 * Connecting to MASTER 127.0.0.1:7001
[17744] 22 May 22:07:18.412 * MASTER <-> REPLICA sync started
[17744] 22 May 22:07:20.441 * Non blocking connect for SYNC fired the event.
[17744] 22 May 22:07:20.441 # Sending command to master in replication handshake: -Writing to master: 找不到指定的模块。
[17744] 22 May 22:07:20.632 * Connecting to MASTER 127.0.0.1:7001
[17744] 22 May 22:07:20.632 * MASTER <-> REPLICA sync started
[17744] 22 May 22:07:22.661 * Non blocking connect for SYNC fired the event.
[17744] 22 May 22:07:22.661 # Sending command to master in replication handshake: -Writing to master: 找不到指定的模块。
[17744] 22 May 22:07:22.852 * Connecting to MASTER 127.0.0.1:7001
[17744] 22 May 22:07:22.852 * MASTER <-> REPLICA sync started
[17744] 22 May 22:07:24.311 * REPLICAOF 127.0.0.1:7002 enabled (user request from 'id=5 addr=127.0.0.1:65456 fd=9 name=sentinel-998f8853-cmd age=441 idle=0 flags=x db=0 sub=0 psub=0 multi=3 qbuf=280 qbuf-free=32488 obl=36 oll=0 omem=0 events=r cmd=exec')
[17744] 22 May 22:07:24.324 # CONFIG REWRITE executed with success.
[17744] 22 May 22:07:25.075 * Connecting to MASTER 127.0.0.1:7002
[17744] 22 May 22:07:25.075 * MASTER <-> REPLICA sync started
[17744] 22 May 22:07:25.076 * Non blocking connect for SYNC fired the event.
[17744] 22 May 22:07:25.076 * Master replied to PING, replication can continue...
[17744] 22 May 22:07:25.077 * Trying a partial resynchronization (request fb437967af3d1b63e3be97fe62b4853e954a20f1:70956).
[17744] 22 May 22:07:25.077 * Successful partial resynchronization with master.
[17744] 22 May 22:07:25.078 # Master replication ID changed to 710dc6e5b7f14e9a40fdf25833a7809e5ac18791
[17744] 22 May 22:07:25.078 * MASTER <-> REPLICA sync: Master accepted a Partial Resynchronization.

```



哨兵7501节点：

```sh

C:\Program Files\redis>redis-server.exe sentinel1/sentinel.conf --sentinel
[15520] 22 May 22:00:03.413 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
[15520] 22 May 22:00:03.413 # Redis version=5.0.14.1, bits=64, commit=ec77f72d, modified=0, pid=15520, just started
[15520] 22 May 22:00:03.413 # Configuration loaded
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 5.0.14.1 (ec77f72d/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in sentinel mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 7501
 |    `-._   `._    /     _.-'    |     PID: 15520
  `-._    `-._  `-./  _.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

[15520] 22 May 22:00:03.416 # Sentinel ID is 998f885399f6bcf3da404d8d332653d6c4137eff
[15520] 22 May 22:00:03.416 # +monitor master mymaster 127.0.0.1 7001 quorum 2
[15520] 22 May 22:00:03.418 * +slave slave 127.0.0.1:7002 127.0.0.1 7002 @ mymaster 127.0.0.1 7001
[15520] 22 May 22:00:03.425 * +slave slave 127.0.0.1:7003 127.0.0.1 7003 @ mymaster 127.0.0.1 7001
[15520] 22 May 22:01:25.957 * +sentinel sentinel 0fa1387c46f90018527a3fa15771de5091da855a 127.0.0.1 7502 @ mymaster 127.0.0.1 7001
[15520] 22 May 22:02:21.870 * +sentinel sentinel b7fd68ca34e48df8850e596a35935a67a324aeae 127.0.0.1 7503 @ mymaster 127.0.0.1 7001
[15520] 22 May 22:07:23.170 # +sdown master mymaster 127.0.0.1 7001
[15520] 22 May 22:07:23.233 # +odown master mymaster 127.0.0.1 7001 #quorum 2/2
[15520] 22 May 22:07:23.233 # +new-epoch 1
[15520] 22 May 22:07:23.233 # +try-failover master mymaster 127.0.0.1 7001
[15520] 22 May 22:07:23.241 # +vote-for-leader 998f885399f6bcf3da404d8d332653d6c4137eff 1
[15520] 22 May 22:07:23.252 # b7fd68ca34e48df8850e596a35935a67a324aeae voted for 998f885399f6bcf3da404d8d332653d6c4137eff 1
[15520] 22 May 22:07:23.254 # 0fa1387c46f90018527a3fa15771de5091da855a voted for 998f885399f6bcf3da404d8d332653d6c4137eff 1
[15520] 22 May 22:07:23.297 # +elected-leader master mymaster 127.0.0.1 7001
[15520] 22 May 22:07:23.297 # +failover-state-select-slave master mymaster 127.0.0.1 7001
[15520] 22 May 22:07:23.361 # +selected-slave slave 127.0.0.1:7002 127.0.0.1 7002 @ mymaster 127.0.0.1 7001
[15520] 22 May 22:07:23.361 * +failover-state-send-slaveof-noone slave 127.0.0.1:7002 127.0.0.1 7002 @ mymaster 127.0.0.1 7001
[15520] 22 May 22:07:23.439 * +failover-state-wait-promotion slave 127.0.0.1:7002 127.0.0.1 7002 @ mymaster 127.0.0.1 7001
[15520] 22 May 22:07:24.254 # +promoted-slave slave 127.0.0.1:7002 127.0.0.1 7002 @ mymaster 127.0.0.1 7001
[15520] 22 May 22:07:24.254 # +failover-state-reconf-slaves master mymaster 127.0.0.1 7001
[15520] 22 May 22:07:24.311 * +slave-reconf-sent slave 127.0.0.1:7003 127.0.0.1 7003 @ mymaster 127.0.0.1 7001
[15520] 22 May 22:07:25.282 * +slave-reconf-inprog slave 127.0.0.1:7003 127.0.0.1 7003 @ mymaster 127.0.0.1 7001
[15520] 22 May 22:07:25.282 * +slave-reconf-done slave 127.0.0.1:7003 127.0.0.1 7003 @ mymaster 127.0.0.1 7001
[15520] 22 May 22:07:25.362 # +failover-end master mymaster 127.0.0.1 7001
[15520] 22 May 22:07:25.362 # +switch-master mymaster 127.0.0.1 7001 127.0.0.1 7002
[15520] 22 May 22:07:25.369 * +slave slave 127.0.0.1:7003 127.0.0.1 7003 @ mymaster 127.0.0.1 7002
[15520] 22 May 22:07:25.374 * +slave slave 127.0.0.1:7001 127.0.0.1 7001 @ mymaster 127.0.0.1 7002
[15520] 22 May 22:07:30.438 # +sdown slave 127.0.0.1:7001 127.0.0.1 7001 @ mymaster 127.0.0.1 7002
[15520] 22 May 22:15:14.153 # -sdown slave 127.0.0.1:7001 127.0.0.1 7001 @ mymaster 127.0.0.1 7002

```



哨兵7502节点：

```sh

C:\Program Files\redis>redis-server.exe sentinel2/sentinel.conf --sentinel
[11912] 22 May 22:01:23.955 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
[11912] 22 May 22:01:23.955 # Redis version=5.0.14.1, bits=64, commit=ec77f72d, modified=0, pid=11912, just started
[11912] 22 May 22:01:23.955 # Configuration loaded
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 5.0.14.1 (ec77f72d/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in sentinel mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 7502
 |    `-._   `._    /     _.-'    |     PID: 11912
  `-._    `-._  `-./  _.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

[11912] 22 May 22:01:23.958 # Sentinel ID is 0fa1387c46f90018527a3fa15771de5091da855a
[11912] 22 May 22:01:23.958 # +monitor master mymaster 127.0.0.1 7001 quorum 2
[11912] 22 May 22:01:23.968 * +slave slave 127.0.0.1:7002 127.0.0.1 7002 @ mymaster 127.0.0.1 7001
[11912] 22 May 22:01:23.973 * +slave slave 127.0.0.1:7003 127.0.0.1 7003 @ mymaster 127.0.0.1 7001
[11912] 22 May 22:01:24.904 * +sentinel sentinel 998f885399f6bcf3da404d8d332653d6c4137eff 127.0.0.1 7501 @ mymaster 127.0.0.1 7001
[11912] 22 May 22:02:21.870 * +sentinel sentinel b7fd68ca34e48df8850e596a35935a67a324aeae 127.0.0.1 7503 @ mymaster 127.0.0.1 7001
[11912] 22 May 22:07:23.233 # +sdown master mymaster 127.0.0.1 7001
[11912] 22 May 22:07:23.247 # +new-epoch 1
[11912] 22 May 22:07:23.254 # +vote-for-leader 998f885399f6bcf3da404d8d332653d6c4137eff 1
[11912] 22 May 22:07:23.345 # +odown master mymaster 127.0.0.1 7001 #quorum 3/2
[11912] 22 May 22:07:23.345 # Next failover delay: I will not start a failover before Sun May 22 22:09:23 2022
[11912] 22 May 22:07:24.311 # +config-update-from sentinel 998f885399f6bcf3da404d8d332653d6c4137eff 127.0.0.1 7501 @ mymaster 127.0.0.1 7001
[11912] 22 May 22:07:24.311 # +switch-master mymaster 127.0.0.1 7001 127.0.0.1 7002
[11912] 22 May 22:07:24.313 * +slave slave 127.0.0.1:7003 127.0.0.1 7003 @ mymaster 127.0.0.1 7002
[11912] 22 May 22:07:24.317 * +slave slave 127.0.0.1:7001 127.0.0.1 7001 @ mymaster 127.0.0.1 7002
[11912] 22 May 22:07:29.360 # +sdown slave 127.0.0.1:7001 127.0.0.1 7001 @ mymaster 127.0.0.1 7002
[11912] 22 May 22:15:13.071 # -sdown slave 127.0.0.1:7001 127.0.0.1 7001 @ mymaster 127.0.0.1 7002

```



哨兵7503节点：

```sh

C:\Program Files\redis>redis-server.exe sentinel3/sentinel.conf --sentinel
[18740] 22 May 22:02:19.809 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
[18740] 22 May 22:02:19.810 # Redis version=5.0.14.1, bits=64, commit=ec77f72d, modified=0, pid=18740, just started
[18740] 22 May 22:02:19.810 # Configuration loaded
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 5.0.14.1 (ec77f72d/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in sentinel mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 7503
 |    `-._   `._    /     _.-'    |     PID: 18740
  `-._    `-._  `-./  _.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

[18740] 22 May 22:02:19.818 # Sentinel ID is b7fd68ca34e48df8850e596a35935a67a324aeae
[18740] 22 May 22:02:19.818 # +monitor master mymaster 127.0.0.1 7001 quorum 2
[18740] 22 May 22:02:19.819 * +slave slave 127.0.0.1:7002 127.0.0.1 7002 @ mymaster 127.0.0.1 7001
[18740] 22 May 22:02:19.825 * +slave slave 127.0.0.1:7003 127.0.0.1 7003 @ mymaster 127.0.0.1 7001
[18740] 22 May 22:02:20.026 * +sentinel sentinel 998f885399f6bcf3da404d8d332653d6c4137eff 127.0.0.1 7501 @ mymaster 127.0.0.1 7001
[18740] 22 May 22:02:21.248 * +sentinel sentinel 0fa1387c46f90018527a3fa15771de5091da855a 127.0.0.1 7502 @ mymaster 127.0.0.1 7001
[18740] 22 May 22:07:23.170 # +sdown master mymaster 127.0.0.1 7001
[18740] 22 May 22:07:23.246 # +new-epoch 1
[18740] 22 May 22:07:23.252 # +vote-for-leader 998f885399f6bcf3da404d8d332653d6c4137eff 1
[18740] 22 May 22:07:23.281 # +odown master mymaster 127.0.0.1 7001 #quorum 2/2
[18740] 22 May 22:07:23.281 # Next failover delay: I will not start a failover before Sun May 22 22:09:23 2022
[18740] 22 May 22:07:24.311 # +config-update-from sentinel 998f885399f6bcf3da404d8d332653d6c4137eff 127.0.0.1 7501 @ mymaster 127.0.0.1 7001
[18740] 22 May 22:07:24.312 # +switch-master mymaster 127.0.0.1 7001 127.0.0.1 7002
[18740] 22 May 22:07:24.313 * +slave slave 127.0.0.1:7003 127.0.0.1 7003 @ mymaster 127.0.0.1 7002
[18740] 22 May 22:07:24.317 * +slave slave 127.0.0.1:7001 127.0.0.1 7001 @ mymaster 127.0.0.1 7002
[18740] 22 May 22:07:29.328 # +sdown slave 127.0.0.1:7001 127.0.0.1 7001 @ mymaster 127.0.0.1 7002
[18740] 22 May 22:15:13.023 # -sdown slave 127.0.0.1:7001 127.0.0.1 7001 @ mymaster 127.0.0.1 7002
[18740] 22 May 22:15:22.995 * +convert-to-slave slave 127.0.0.1:7001 127.0.0.1 7001 @ mymaster 127.0.0.1 7002

```





## windows哨兵模式启动脚本

```sh
start "redis-7001" redis-server.exe redis1/redis.conf
start "redis-7002" redis-server.exe redis2/redis.conf
start "redis-7003" redis-server.exe redis3/redis.conf
timeout /nobreak /t 3
start "redis-sentinel-7501" redis-server.exe sentinel1/sentinel.conf --sentinel
start "redis-sentinel-7502" redis-server.exe sentinel2/sentinel.conf --sentinel
start "redis-sentinel-7503" redis-server.exe sentinel3/sentinel.conf --sentinel
```





## java代码操作redis哨兵集群

在Sentinel集群监管下的Redis主从集群，其节点会因为自动故障转移而发生变化，Redis的客户端必须感知这种变化，及时更新连接信息。Spring的RedisTemplate底层利用lettuce实现了节点的感知和自动切换。

### 引入依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```



### 配置

在配置文件application.yml中指定redis的sentinel相关信息

```yaml
spring:
  redis:
    sentinel:
      master: mymaster
      nodes:
        - 127.0.0.1:7501
        - 127.0.0.1:7502
        - 127.0.0.1:7503
```



### 配置读写分离

```java
@Configuration
public class RedisConfig
{

    @Bean
    public LettuceClientConfigurationBuilderCustomizer lettuceClientConfigurationBuilderCustomizer()
    {
        
        return clientConfigurationBuilder -> clientConfigurationBuilder.readFrom(ReadFrom.REPLICA_PREFERRED);
    }
}
```

读写策略：

- MASTER：从主节点读取
- MASTER_PREFERRED：优先从master节点读取，master不可用才读取replica
- REPLICA：从slave（replica）节点读取
- REPLICA _PREFERRED：优先从slave（replica）节点读取，所有的slave都不可用才读取master



### RedisTestController

```java
@RestController
public class RedisTestController
{
    @Resource
    private StringRedisTemplate stringRedisTemplate;

    @GetMapping("set/{key}/{value}")
    public Boolean set(@PathVariable String key, @PathVariable String value)
    {
        stringRedisTemplate.opsForValue().set(key, value);
        return true;
    }

    @GetMapping("get/{key}")
    public String get(@PathVariable String key)
    {
        return stringRedisTemplate.opsForValue().get(key);
    }
}

```



### 结果

向redis里存一个数：

http://localhost:8080/set/a/1267

控制台结果：

```sh
2022-05-23 13:21:04.135 DEBUG 11808 --- [o-8080-Acceptor] o.apache.tomcat.util.threads.LimitLatch  : Counting up[http-nio-8080-Acceptor] latch=1
2022-05-23 13:21:04.136 DEBUG 11808 --- [o-8080-Acceptor] o.apache.tomcat.util.threads.LimitLatch  : Counting up[http-nio-8080-Acceptor] latch=2
2022-05-23 13:21:04.140 DEBUG 11808 --- [nio-8080-exec-4] o.a.coyote.http11.Http11InputBuffer      : Before fill(): parsingHeader: [true], parsingRequestLine: [true], parsingRequestLinePhase: [0], parsingRequestLineStart: [0], byteBuffer.position(): [0], byteBuffer.limit(): [0], end: [938]
2022-05-23 13:21:04.140 DEBUG 11808 --- [nio-8080-exec-4] o.a.tomcat.util.net.SocketWrapperBase    : Socket: [org.apache.tomcat.util.net.NioEndpoint$NioSocketWrapper@78e2c444:org.apache.tomcat.util.net.NioChannel@3489c5de:java.nio.channels.SocketChannel[connected local=/[0:0:0:0:0:0:0:1]:8080 remote=/[0:0:0:0:0:0:0:1]:60091]], Read from buffer: [0]
2022-05-23 13:21:04.140 DEBUG 11808 --- [nio-8080-exec-4] org.apache.tomcat.util.net.NioEndpoint   : Socket: [org.apache.tomcat.util.net.NioEndpoint$NioSocketWrapper@78e2c444:org.apache.tomcat.util.net.NioChannel@3489c5de:java.nio.channels.SocketChannel[connected local=/[0:0:0:0:0:0:0:1]:8080 remote=/[0:0:0:0:0:0:0:1]:60091]], Read direct from socket: [917]
2022-05-23 13:21:04.140 DEBUG 11808 --- [nio-8080-exec-4] o.a.coyote.http11.Http11InputBuffer      : Received [GET /set/a/1267 HTTP/1.1
Host: localhost:8080
Connection: keep-alive
sec-ch-ua: " Not A;Brand";v="99", "Chromium";v="101", "Microsoft Edge";v="101"
sec-ch-ua-mobile: ?0
sec-ch-ua-platform: "Windows"
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/101.0.4951.64 Safari/537.36 Edg/101.0.1210.53
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Sec-Fetch-Site: none
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Accept-Encoding: gzip, deflate, br
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6
Cookie: Idea-2347e683=7bef4e77-fa42-4f63-b13f-9d49fe35fcf9; mbox=session#a01c13ff0816407685902a031e6d50bd#1644071823|PC#a01c13ff0816407685902a031e6d50bd.32_0#1678249975; _ga=GA1.1.2061233658.1650939704

]
2022-05-23 13:21:04.141 DEBUG 11808 --- [nio-8080-exec-4] o.a.t.util.http.Rfc6265CookieProcessor   : Cookies: Parsing b[]: Idea-2347e683=7bef4e77-fa42-4f63-b13f-9d49fe35fcf9; mbox=session#a01c13ff0816407685902a031e6d50bd#1644071823|PC#a01c13ff0816407685902a031e6d50bd.32_0#1678249975; _ga=GA1.1.2061233658.1650939704
2022-05-23 13:21:04.141 DEBUG 11808 --- [nio-8080-exec-4] o.a.c.authenticator.AuthenticatorBase    : Security checking request GET /set/a/1267
2022-05-23 13:21:04.141 DEBUG 11808 --- [nio-8080-exec-4] org.apache.catalina.realm.RealmBase      :   No applicable constraints defined
2022-05-23 13:21:04.141 DEBUG 11808 --- [nio-8080-exec-4] o.a.c.authenticator.AuthenticatorBase    : Not subject to any constraint
2022-05-23 13:21:04.142 DEBUG 11808 --- [nio-8080-exec-4] org.apache.tomcat.util.http.Parameters   : Set encoding to UTF-8
2022-05-23 13:21:04.142 DEBUG 11808 --- [nio-8080-exec-4] o.s.web.servlet.DispatcherServlet        : GET "/set/a/1267", parameters={}
2022-05-23 13:21:04.142 DEBUG 11808 --- [nio-8080-exec-4] s.w.s.m.m.a.RequestMappingHandlerMapping : Mapped to mao.redis_sentinel_cluster.controller.RedisTestController#set(String, String)
2022-05-23 13:21:04.143 DEBUG 11808 --- [nio-8080-exec-4] o.s.d.redis.core.RedisConnectionUtils    : Fetching Redis Connection from RedisConnectionFactory
2022-05-23 13:21:04.145 DEBUG 11808 --- [nio-8080-exec-4] io.lettuce.core.RedisChannelHandler      : dispatching command AsyncCommand [type=SET, output=StatusOutput [output=null, error='null'], commandType=io.lettuce.core.protocol.Command]
2022-05-23 13:21:04.145 DEBUG 11808 --- [nio-8080-exec-4] i.l.c.m.MasterReplicaConnectionProvider  : getConnectionAsync(WRITE)
2022-05-23 13:21:04.145 DEBUG 11808 --- [nio-8080-exec-4] io.lettuce.core.RedisChannelHandler      : dispatching command AsyncCommand [type=SET, output=StatusOutput [output=null, error='null'], commandType=io.lettuce.core.protocol.Command]
2022-05-23 13:21:04.145 DEBUG 11808 --- [nio-8080-exec-4] i.lettuce.core.protocol.DefaultEndpoint  : [channel=0x46c53bca, /127.0.0.1:60024 -> /127.0.0.1:7001, epid=0xa] write() writeAndFlush command AsyncCommand [type=SET, output=StatusOutput [output=null, error='null'], commandType=io.lettuce.core.protocol.Command]
2022-05-23 13:21:04.145 DEBUG 11808 --- [nio-8080-exec-4] i.lettuce.core.protocol.DefaultEndpoint  : [channel=0x46c53bca, /127.0.0.1:60024 -> /127.0.0.1:7001, epid=0xa] write() done
2022-05-23 13:21:04.145 DEBUG 11808 --- [oEventLoop-4-10] io.lettuce.core.protocol.CommandHandler  : [channel=0x46c53bca, /127.0.0.1:60024 -> /127.0.0.1:7001, epid=0xa, chid=0xa] write(ctx, AsyncCommand [type=SET, output=StatusOutput [output=null, error='null'], commandType=io.lettuce.core.protocol.Command], promise)
2022-05-23 13:21:04.146 DEBUG 11808 --- [oEventLoop-4-10] io.lettuce.core.protocol.CommandEncoder  : [channel=0x46c53bca, /127.0.0.1:60024 -> /127.0.0.1:7001] writing command AsyncCommand [type=SET, output=StatusOutput [output=null, error='null'], commandType=io.lettuce.core.protocol.Command]
2022-05-23 13:21:04.146 DEBUG 11808 --- [oEventLoop-4-10] io.lettuce.core.protocol.CommandHandler  : [channel=0x46c53bca, /127.0.0.1:60024 -> /127.0.0.1:7001, epid=0xa, chid=0xa] Received: 5 bytes, 1 commands in the stack
2022-05-23 13:21:04.146 DEBUG 11808 --- [oEventLoop-4-10] io.lettuce.core.protocol.CommandHandler  : [channel=0x46c53bca, /127.0.0.1:60024 -> /127.0.0.1:7001, epid=0xa, chid=0xa] Stack contains: 1 commands
2022-05-23 13:21:04.146 DEBUG 11808 --- [oEventLoop-4-10] i.l.core.protocol.RedisStateMachine      : Decode done, empty stack: true
2022-05-23 13:21:04.146 DEBUG 11808 --- [oEventLoop-4-10] io.lettuce.core.protocol.CommandHandler  : [channel=0x46c53bca, /127.0.0.1:60024 -> /127.0.0.1:7001, epid=0xa, chid=0xa] Completing command AsyncCommand [type=SET, output=StatusOutput [output=OK, error='null'], commandType=io.lettuce.core.protocol.Command]
2022-05-23 13:21:04.146 DEBUG 11808 --- [nio-8080-exec-4] o.s.d.redis.core.RedisConnectionUtils    : Closing Redis Connection.
2022-05-23 13:21:04.147 DEBUG 11808 --- [nio-8080-exec-4] m.m.a.RequestResponseBodyMethodProcessor : Using 'application/json;q=0.8', given [text/html, application/xhtml+xml, image/webp, image/apng, application/xml;q=0.9, application/signed-exchange;v=b3;q=0.9, */*;q=0.8] and supported [application/json, application/*+json, application/json, application/*+json]
2022-05-23 13:21:04.147 DEBUG 11808 --- [nio-8080-exec-4] m.m.a.RequestResponseBodyMethodProcessor : Writing [true]
2022-05-23 13:21:04.153 DEBUG 11808 --- [nio-8080-exec-4] o.s.web.servlet.DispatcherServlet        : Completed 200 OK
2022-05-23 13:21:04.153 DEBUG 11808 --- [nio-8080-exec-4] o.a.coyote.http11.Http11InputBuffer      : Before fill(): parsingHeader: [true], parsingRequestLine: [true], parsingRequestLinePhase: [0], parsingRequestLineStart: [0], byteBuffer.position(): [0], byteBuffer.limit(): [0], end: [917]
2022-05-23 13:21:04.153 DEBUG 11808 --- [nio-8080-exec-4] o.a.tomcat.util.net.SocketWrapperBase    : Socket: [org.apache.tomcat.util.net.NioEndpoint$NioSocketWrapper@78e2c444:org.apache.tomcat.util.net.NioChannel@3489c5de:java.nio.channels.SocketChannel[connected local=/[0:0:0:0:0:0:0:1]:8080 remote=/[0:0:0:0:0:0:0:1]:60091]], Read from buffer: [0]
2022-05-23 13:21:04.153 DEBUG 11808 --- [nio-8080-exec-4] org.apache.tomcat.util.net.NioEndpoint   : Socket: [org.apache.tomcat.util.net.NioEndpoint$NioSocketWrapper@78e2c444:org.apache.tomcat.util.net.NioChannel@3489c5de:java.nio.channels.SocketChannel[connected local=/[0:0:0:0:0:0:0:1]:8080 remote=/[0:0:0:0:0:0:0:1]:60091]], Read direct from socket: [0]
2022-05-23 13:21:04.153 DEBUG 11808 --- [nio-8080-exec-4] o.a.coyote.http11.Http11InputBuffer      : Received []
2022-05-23 13:21:04.154 DEBUG 11808 --- [nio-8080-exec-4] o.apache.coyote.http11.Http11Processor   : Socket: [org.apache.tomcat.util.net.NioEndpoint$NioSocketWrapper@78e2c444:org.apache.tomcat.util.net.NioChannel@3489c5de:java.nio.channels.SocketChannel[connected local=/[0:0:0:0:0:0:0:1]:8080 remote=/[0:0:0:0:0:0:0:1]:60091]], Status in: [OPEN_READ], State out: [OPEN]
2022-05-23 13:21:04.154 DEBUG 11808 --- [nio-8080-exec-4] org.apache.tomcat.util.net.NioEndpoint   : Registered read interest for [org.apache.tomcat.util.net.NioEndpoint$NioSocketWrapper@78e2c444:org.apache.tomcat.util.net.NioChannel@3489c5de:java.nio.channels.SocketChannel[connected local=/[0:0:0:0:0:0:0:1]:8080 remote=/[0:0:0:0:0:0:0:1]:60091]]

```



向redis里取一个数：

http://localhost:8080/get/a

结果：

```sh
2022-05-23 13:22:37.762 DEBUG 11808 --- [o-8080-Acceptor] o.apache.tomcat.util.threads.LimitLatch  : Counting up[http-nio-8080-Acceptor] latch=1
2022-05-23 13:22:37.763 DEBUG 11808 --- [o-8080-Acceptor] o.apache.tomcat.util.threads.LimitLatch  : Counting up[http-nio-8080-Acceptor] latch=2
2022-05-23 13:22:37.767 DEBUG 11808 --- [nio-8080-exec-7] o.a.coyote.http11.Http11InputBuffer      : Before fill(): parsingHeader: [true], parsingRequestLine: [true], parsingRequestLinePhase: [0], parsingRequestLineStart: [0], byteBuffer.position(): [0], byteBuffer.limit(): [0], end: [917]
2022-05-23 13:22:37.767 DEBUG 11808 --- [nio-8080-exec-7] o.a.tomcat.util.net.SocketWrapperBase    : Socket: [org.apache.tomcat.util.net.NioEndpoint$NioSocketWrapper@1d57775f:org.apache.tomcat.util.net.NioChannel@3489c5de:java.nio.channels.SocketChannel[connected local=/[0:0:0:0:0:0:0:1]:8080 remote=/[0:0:0:0:0:0:0:1]:60152]], Read from buffer: [0]
2022-05-23 13:22:37.767 DEBUG 11808 --- [nio-8080-exec-7] org.apache.tomcat.util.net.NioEndpoint   : Socket: [org.apache.tomcat.util.net.NioEndpoint$NioSocketWrapper@1d57775f:org.apache.tomcat.util.net.NioChannel@3489c5de:java.nio.channels.SocketChannel[connected local=/[0:0:0:0:0:0:0:1]:8080 remote=/[0:0:0:0:0:0:0:1]:60152]], Read direct from socket: [912]
2022-05-23 13:22:37.767 DEBUG 11808 --- [nio-8080-exec-7] o.a.coyote.http11.Http11InputBuffer      : Received [GET /get/a HTTP/1.1
Host: localhost:8080
Connection: keep-alive
sec-ch-ua: " Not A;Brand";v="99", "Chromium";v="101", "Microsoft Edge";v="101"
sec-ch-ua-mobile: ?0
sec-ch-ua-platform: "Windows"
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/101.0.4951.64 Safari/537.36 Edg/101.0.1210.53
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Sec-Fetch-Site: none
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Accept-Encoding: gzip, deflate, br
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6
Cookie: Idea-2347e683=7bef4e77-fa42-4f63-b13f-9d49fe35fcf9; mbox=session#a01c13ff0816407685902a031e6d50bd#1644071823|PC#a01c13ff0816407685902a031e6d50bd.32_0#1678249975; _ga=GA1.1.2061233658.1650939704

]
2022-05-23 13:22:37.768 DEBUG 11808 --- [nio-8080-exec-7] o.a.t.util.http.Rfc6265CookieProcessor   : Cookies: Parsing b[]: Idea-2347e683=7bef4e77-fa42-4f63-b13f-9d49fe35fcf9; mbox=session#a01c13ff0816407685902a031e6d50bd#1644071823|PC#a01c13ff0816407685902a031e6d50bd.32_0#1678249975; _ga=GA1.1.2061233658.1650939704
2022-05-23 13:22:37.768 DEBUG 11808 --- [nio-8080-exec-7] o.a.c.authenticator.AuthenticatorBase    : Security checking request GET /get/a
2022-05-23 13:22:37.768 DEBUG 11808 --- [nio-8080-exec-7] org.apache.catalina.realm.RealmBase      :   No applicable constraints defined
2022-05-23 13:22:37.768 DEBUG 11808 --- [nio-8080-exec-7] o.a.c.authenticator.AuthenticatorBase    : Not subject to any constraint
2022-05-23 13:22:37.769 DEBUG 11808 --- [nio-8080-exec-7] org.apache.tomcat.util.http.Parameters   : Set encoding to UTF-8
2022-05-23 13:22:37.769 DEBUG 11808 --- [nio-8080-exec-7] o.s.web.servlet.DispatcherServlet        : GET "/get/a", parameters={}
2022-05-23 13:22:37.769 DEBUG 11808 --- [nio-8080-exec-7] s.w.s.m.m.a.RequestMappingHandlerMapping : Mapped to mao.redis_sentinel_cluster.controller.RedisTestController#get(String)
2022-05-23 13:22:37.770 DEBUG 11808 --- [nio-8080-exec-7] o.s.d.redis.core.RedisConnectionUtils    : Fetching Redis Connection from RedisConnectionFactory
2022-05-23 13:22:37.770 DEBUG 11808 --- [nio-8080-exec-7] io.lettuce.core.RedisChannelHandler      : dispatching command AsyncCommand [type=GET, output=ValueOutput [output=null, error='null'], commandType=io.lettuce.core.protocol.Command]
2022-05-23 13:22:37.770 DEBUG 11808 --- [nio-8080-exec-7] i.l.c.m.MasterReplicaConnectionProvider  : getConnectionAsync(READ)
2022-05-23 13:22:37.770 DEBUG 11808 --- [nio-8080-exec-7] io.lettuce.core.RedisChannelHandler      : dispatching command AsyncCommand [type=GET, output=ValueOutput [output=null, error='null'], commandType=io.lettuce.core.protocol.Command]
2022-05-23 13:22:37.770 DEBUG 11808 --- [nio-8080-exec-7] i.lettuce.core.protocol.DefaultEndpoint  : [channel=0x9ae76d5f, /127.0.0.1:60022 -> /127.0.0.1:7002, epid=0x8] write() writeAndFlush command AsyncCommand [type=GET, output=ValueOutput [output=null, error='null'], commandType=io.lettuce.core.protocol.Command]
2022-05-23 13:22:37.771 DEBUG 11808 --- [nio-8080-exec-7] i.lettuce.core.protocol.DefaultEndpoint  : [channel=0x9ae76d5f, /127.0.0.1:60022 -> /127.0.0.1:7002, epid=0x8] write() done
2022-05-23 13:22:37.771 DEBUG 11808 --- [ioEventLoop-4-8] io.lettuce.core.protocol.CommandHandler  : [channel=0x9ae76d5f, /127.0.0.1:60022 -> /127.0.0.1:7002, epid=0x8, chid=0x8] write(ctx, AsyncCommand [type=GET, output=ValueOutput [output=null, error='null'], commandType=io.lettuce.core.protocol.Command], promise)
2022-05-23 13:22:37.772 DEBUG 11808 --- [ioEventLoop-4-8] io.lettuce.core.protocol.CommandEncoder  : [channel=0x9ae76d5f, /127.0.0.1:60022 -> /127.0.0.1:7002] writing command AsyncCommand [type=GET, output=ValueOutput [output=null, error='null'], commandType=io.lettuce.core.protocol.Command]
2022-05-23 13:22:37.772 DEBUG 11808 --- [ioEventLoop-4-8] io.lettuce.core.protocol.CommandHandler  : [channel=0x9ae76d5f, /127.0.0.1:60022 -> /127.0.0.1:7002, epid=0x8, chid=0x8] Received: 10 bytes, 1 commands in the stack
2022-05-23 13:22:37.772 DEBUG 11808 --- [ioEventLoop-4-8] io.lettuce.core.protocol.CommandHandler  : [channel=0x9ae76d5f, /127.0.0.1:60022 -> /127.0.0.1:7002, epid=0x8, chid=0x8] Stack contains: 1 commands
2022-05-23 13:22:37.772 DEBUG 11808 --- [ioEventLoop-4-8] i.l.core.protocol.RedisStateMachine      : Decode done, empty stack: true
2022-05-23 13:22:37.772 DEBUG 11808 --- [ioEventLoop-4-8] io.lettuce.core.protocol.CommandHandler  : [channel=0x9ae76d5f, /127.0.0.1:60022 -> /127.0.0.1:7002, epid=0x8, chid=0x8] Completing command AsyncCommand [type=GET, output=ValueOutput [output=[B@5679d7e, error='null'], commandType=io.lettuce.core.protocol.Command]
2022-05-23 13:22:37.772 DEBUG 11808 --- [nio-8080-exec-7] o.s.d.redis.core.RedisConnectionUtils    : Closing Redis Connection.
2022-05-23 13:22:37.773 DEBUG 11808 --- [nio-8080-exec-7] m.m.a.RequestResponseBodyMethodProcessor : Using 'text/html', given [text/html, application/xhtml+xml, image/webp, image/apng, application/xml;q=0.9, application/signed-exchange;v=b3;q=0.9, */*;q=0.8] and supported [text/plain, */*, text/plain, */*, application/json, application/*+json, application/json, application/*+json]
2022-05-23 13:22:37.773 DEBUG 11808 --- [nio-8080-exec-7] m.m.a.RequestResponseBodyMethodProcessor : Writing ["1267"]
2022-05-23 13:22:37.774 DEBUG 11808 --- [nio-8080-exec-7] o.s.web.servlet.DispatcherServlet        : Completed 200 OK
2022-05-23 13:22:37.774 DEBUG 11808 --- [nio-8080-exec-7] o.a.coyote.http11.Http11InputBuffer      : Before fill(): parsingHeader: [true], parsingRequestLine: [true], parsingRequestLinePhase: [0], parsingRequestLineStart: [0], byteBuffer.position(): [0], byteBuffer.limit(): [0], end: [912]
2022-05-23 13:22:37.774 DEBUG 11808 --- [nio-8080-exec-7] o.a.tomcat.util.net.SocketWrapperBase    : Socket: [org.apache.tomcat.util.net.NioEndpoint$NioSocketWrapper@1d57775f:org.apache.tomcat.util.net.NioChannel@3489c5de:java.nio.channels.SocketChannel[connected local=/[0:0:0:0:0:0:0:1]:8080 remote=/[0:0:0:0:0:0:0:1]:60152]], Read from buffer: [0]
2022-05-23 13:22:37.774 DEBUG 11808 --- [nio-8080-exec-7] org.apache.tomcat.util.net.NioEndpoint   : Socket: [org.apache.tomcat.util.net.NioEndpoint$NioSocketWrapper@1d57775f:org.apache.tomcat.util.net.NioChannel@3489c5de:java.nio.channels.SocketChannel[connected local=/[0:0:0:0:0:0:0:1]:8080 remote=/[0:0:0:0:0:0:0:1]:60152]], Read direct from socket: [0]
2022-05-23 13:22:37.774 DEBUG 11808 --- [nio-8080-exec-7] o.a.coyote.http11.Http11InputBuffer      : Received []
2022-05-23 13:22:37.774 DEBUG 11808 --- [nio-8080-exec-7] o.apache.coyote.http11.Http11Processor   : Socket: [org.apache.tomcat.util.net.NioEndpoint$NioSocketWrapper@1d57775f:org.apache.tomcat.util.net.NioChannel@3489c5de:java.nio.channels.SocketChannel[connected local=/[0:0:0:0:0:0:0:1]:8080 remote=/[0:0:0:0:0:0:0:1]:60152]], Status in: [OPEN_READ], State out: [OPEN]
2022-05-23 13:22:37.775 DEBUG 11808 --- [nio-8080-exec-7] org.apache.tomcat.util.net.NioEndpoint   : Registered read interest for [org.apache.tomcat.util.net.NioEndpoint$NioSocketWrapper@1d57775f:org.apache.tomcat.util.net.NioChannel@3489c5de:java.nio.channels.SocketChannel[connected local=/[0:0:0:0:0:0:0:1]:8080 remote=/[0:0:0:0:0:0:0:1]:60152]]

```







# redis分片集群

主从和哨兵可以解决高可用、高并发读的问题。但是依然有两个问题没有解决：

- 海量数据存储问题
- 高并发写的问题



## 特点

- 集群中有多个master，每个master保存不同数据
- 每个master都可以有多个slave节点
- master之间通过ping监测彼此健康状态
- 客户端请求可以访问集群任意节点，最终都会被转发到正确节点



## 集群步骤

### 1. 准备

