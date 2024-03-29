---
title: redis主从之全量复制及增量复制
tags: []
id: '62'
categories:
  - - uncategorized
date: 2020-09-23 06:42:20
---

> 在之前我写了一篇docker实现redis主从复制的文章，点击进入

对于主从复制的好处，在上篇文章我也写了，下面说一下注意事项。

**注意事项**：

*   安全
    
    对于数据比较重要的节点，主节点会通过设置`requirepass`参数进行密码验证，这时候所有的客户端访问必须使用`auth`命令进行验证。从节点与主节点的复制链接是通过一个特殊标识的客户端来完成。因此需要配置从节点的`masterauth`参数与主节点密码保持一致，这样从节点才可以正确地链接到主节点并发起复制流程。
    
*   从节点只读
    
    默认情况下`slave-read-only=yes`配置为只读，由于复制只能从主节点到从节点，对于从节点的任何修改主节点都无法感知，修改从节点会造成主从数据不一致。因此没必要就不要动这个配置。
    
*   网络延迟问题
    
    主从节点一般部署在不同机器上，复制时的网络延迟就成为需要考虑的问题，redis为我们提供了`repl-disable-tcp-nodelay`参数用于控制是否关闭 tcp nodelay，默认是关闭的，说明如下：
    
    > 当**关闭**时，主节点产生的命令数据无论大小都会及时地发送给从节点，这样主从之间延迟将会变小，但增加了网络宽带的消耗。适用于主从之间的网络环境较好的场景。
    > 
    > 当**开启**时，主节点会合并较小的TCP数据包从而节省宽带。默认发送时间间隔取决于Linux的内核，一般默认为40ms。这种配置节省了宽带但增大主从之间的延迟。适用于主从网络环境复杂或宽带紧张的场景。
    

部署主从节点时需要考虑网络延迟、宽带使用率、防灾级别等因素，如要求低延迟时，建议同机房部署并关闭`repl-disable-tcp-nodelay`，如考虑容灾性，可以跨机房部署并开启`repl-disable-tcp-nodelay`。

## 拓扑图

### 一主一从

```

graph TD

A[Redis-master] --> B[Redis-slave]
```

### 一主多从

```
graph TD

A[Redis-master] --> B[Redis-slave]
A[Redis-master] --> C[Redis-slave]
A[Redis-master] --> D[Redis-slave]
```

### 树状主从

```
graph TD

A[Redis-master] --> B[Redis-slave]
A[Redis-master] --> C[Redis-slave]
B[Redis-slave] --> D[Redis-slave]
B[Redis-slave] --> E[Redis-slave]
```

### 原理

```
graph TD

A[slaveof] -->127.0.0.1:6379 B[slave]
B[slave] --> D[保存主节点信息]
D[保存主节点信息] --> E[主从建立socket连接]
E[主从建立socket连接] --> F[发送ping命令]
F[发送ping命令] --> G[权限验证]
G[权限验证] --> H[同步数据集]
H[同步数据集] --> I[命令持续复制]
I[命令持续复制] --> J[master]
```

从上图可以看出来大致分为6个过程：

*   执行slaveof后从节点保存主节点的地址信息便返回，这时候复制流程还没开始。
*   从节点内部通过每秒运行的定时任务维护复制相关逻辑，当定时任务发现存在新的主节点后，会尝试与该节点建立网络连接，从节点会建立一个socket套接字。
*   发送ping命令，检测主从之间网络套接字是否可用，检测主节点是否可用接受处理命令。如果发送 ping 命令后，从节点没有收到主节点的 pong 回复或者超时，比如网络超时或者主节点正在阻塞无法响应命令，从节点会断开复制连接，下次定时任务会发起重连。
*   如果主节点配置了`requirepass`参数，则需要密码认证，从节点必须配置`masterauth`参数保证与主节点相同的密码才能通过验证。
*   主从复制连接正常通信后，对于首次建立复制的场景，主节点会把持有的数据全部发送给从节点，这部分操作是耗时最长的步骤。
*   当主节点把当前的数据同步给从节点后，便完成了复制的建立流程。接下来主节点会持续地把写命令发送给从节点，保证主从数据一致性。

> 主从同步的过程中，从节点会把原来的数据清空。

## 数据同步

同步方式：

*   全量复制
    
    用于初次复制或其它无法进行部分复制的情况，将主节点中的所有数据都发送给从节点。当数据量过大的时候，会造成很大的网络开销。
    
*   部分复制
    
    用于处理在主从复制中因网络闪退等原因造成数据丢失场景，当从节点再次连上主节点，如果条件允许，主节点会补发丢失数据给从节点，因为补发的数据远远小于全量数据，可以有效避免全量复制的过高开销。但需要注意，如果网络中断时间过长，造成主节点没有能够完整地保存中断期间执行的写命令，则无法进行部分复制，仍使用全量复制 。
    

复制偏移量：

*   参与复制的主从节点都会维护自身复制偏移量，主节点在处理完写入命令操作后，会把命令的字节长度做累加记录，统计信息在`info replication`中的`master_repl_offset`指标中。
*   从节点每秒钟上报自身的复制偏移量给主节点，因此主节点也会保存从节点的复制偏移量`slave0:ip=192.168.1.3,port=6379,state=online,offset=116424,lag=0`
*   从节点在接收到主节点发送的命令后，也会累加记录自身的偏移量。统计信息在`info replication`中的`slave_repl_offset`中。

复制积压缓冲区：

*   复制积压缓冲区是保存在主节点上的一个固定长度的队列，默认大小为1MB，当主节点有连接的从节点时被创建，这时主节点响应写命令时，不但会把命令发给从节点，还会写入复制积压缓冲区。
*   在命令传播阶段，主节点除了将写命令发送给从节点，还会发送一份给复制积压缓冲区，作为写命令的备份；除了存储写命令，复制积压缓冲区中还存储了其中 的每个字节对应的复制偏移量(offset) 。由于复制积压缓冲区定长且先进先出，所以它保存的是主节点最近执行的写命令；时间较早的写命令会被挤出缓冲区。

主节点运行ID：

*   每个redis节点启动后都会动态分配一个40位的十六进制字符串为运行ID。运行ID的主要作用是来唯一识别redis节点，比如从节点保存主节点的运行ID识别自已正在复制是哪个主节点。如果只使用ip+port的方式识别主节点，那么主节点重启变更了整体数据集（如替换RDB/AOF文件），从节点再基于偏移量复制数据将是不安全的，因此当运行ID变化后从节点将做全量复制。可以在`info server`命令查看当前节点的运行ID。
*   需要注意的是redis关闭再启动，运行的id会随之变化。

Psync命令：

![](http://qiniu.gaobinzhan.com/2020/06/03/d8f06279c7ec4.png)

*   从节点使用`psync`命令完成部分复制和全量复制功能`psync runid offset`
*   流程说明：
    
    *   从节点(slave)发送psync命令给主节点，参数runid是当前从节点保存的主节点运行id，如果没有则默认值为 ？, 参数offset是当前从节点保存的复制偏移量，如果是第一次参与复制则默认值为-1。
    *   主节点根据`pysnc`参数和自身数据情况决定响应结果：
        
        *   如果回复+FULLRESYNC {runid} {offset}，那么从节点将触发全量复制流程。
        *   如果回复+CONTINUE，从节点将触发部分复制流程。
        *   如果回复-ERR，说明主节点版本低于Redis2.8。

全量复制流程：

![](http://qiniu.gaobinzhan.com/2020/06/03/7351b38954ab9.png)

*   发送psync命令进行数据同步，由于是第一次进行复制，从节点没有复制偏移量和主节点的运行id，所以发送psync ? -1
*   主节点根据psync ? -1解析出当前为全量复制，回复+FULLRESYNC响应(主机会向从机发送 runid 和 offset，因为 slave 并没有对应的 offset，所以是全量复制)
*   从节点接收主节点的响应数据保存运行ID和偏移量offset(从机 slave 会保存 主机master 的基本信息 save masterInfo)
*   主节点收到全量复制的命令后，执行bgsave（异步执行），在后台生成RDB文件（快照），并使用一个缓冲区（称为复制缓冲区）记录从现在开始执行的所有写命令
*   主节点发送RDB文件给从节点，从节点把接收到的RDB文件保存在本地并直接作为从节点的数据文件，接收完RDB后从节点打印相关日志，可以在日志中查看主节点发送的数据量(主机send RDB 发送 RDB 文件给从机)
    
    *   注意！对于数据量较大的主节点，比如生成的RDB文件超过6GB以上时要格外小心。传输文件这一步操作非常耗时，速度取决于主从节点之间网络带宽。
    *   通过细致分析Full resync和MASTER SLAVE这两行日志的时间差，可以算出RDB文件从创建到传输完毕消耗的总时间。如果总时间超过repl-timeout所配置的值 (默认60秒)，从节点将放弃接受RDB文件并清理已经下载的临时文件，导致全量复制失败;针对数据量较大的节点，建议调大repl-timeout参数防止出现全量同步数据超时;
    *   例如对于千兆网卡的机器，网卡带宽理论峰值大约每秒传输100MB,在不考虑其他进程消耗带宽的情况下，6GB的RDB文件至少需要60秒传输时间，默认配置下，极易出现主从数同步超时。
*   对于从节点开始接收RDB快照到接收完成期间，主节点仍然响应读写命令，因此主节点会把这期间写命令数据保存在复制客户端缓冲区内，当从节点加载完RDB文件后，主节点再把缓冲区内的数据发送给从节点，保证主从之间数据致性。(发送缓冲区数据)
*   从节点接收完主节点传送来的全部数据后会清空自身旧数据(刷新旧的数据，从节点在载入主节点的数据之前要先将老数据清除)
*   从节点清空数据后开始加载RDB文件，对于较大的RDB文件，这一步操作依然比较消耗时间，可以通过计算日志之间的实际差来判断加载RDB的总消耗时间(加载 RDB 文件将数据库状态更新至主节点执行bgsave时的数据库状态和缓冲区数据的加载。)
*   从节点成功加载完RDB后，如果当前节点开启了AOF持久化的功能，它会立刻做bgrewriteeaof的操作，为了保证全量复制后AOF持久化文件立刻可用。 通过分析全量复制的所有流程，全量复制是一个非常耗时费力的操作。他的实际开销主要包括：
    
    *   主节点bgsave时间
    *   RDB文件网络传输时间
    *   从节点清空数据时间
    *   从节点加载RDB的时间
    *   可能的AOF重写时间

部分复制流程：

![](http://qiniu.gaobinzhan.com/2020/06/04/57b273afe7d75.png)

*   部分复制是 Redis 2.8 以后出现的，之所以要加入部分复制，是因为全量复制会产生很多问题，比如像上面的时间开销大、无法隔离等问题， Redis 希望能够在主节点出现抖动（相当于断开连接）的时候，可以有一些机制将复制的损失降低到最低
*   当主从节点之间网络出现中断时，如果超过repl-timeout时间，主节点会认为从节点出问题了并断开复制链接（如果网络抖动（连接断开 connection lost））。
*   主从连接中断期间主节点依然响应命令，但因复制链接中断命令无法发送给从节点不过主节点内部存在的复制积压缓存去，依然可以保存一段时间的写命令数据，默认最大缓存1MB(主机master 还是会写 replbackbuffer（复制缓冲区）)
*   当主从节点网络恢复后，从节点会再次连上主节点。(从机slave会继续尝试连接主机)
*   当主从连接恢复后，由于从节点之前保存了自身已复制的偏移量和主节点的运行id。因此会把他们当作psync参数发送给主节点，要求进行部分复制操作。(从机 slave 会把自己当前 runid 和偏移量传输给主机 master，并且执行 pysnc 命令同步)
*   主节点接到psync命令后首先核对参数的runid，如果 master 发现你的偏移量是在缓冲区的范围内，根据参数offset在缓冲区查找复制内内，如果在偏移量之后的数据存在缓存区中，则对从节点发送continue表示可以进行部分复制
*   主节点根据偏移量把复制积压缓冲区里的数据发送给从节点，保证主从复制进入正常状态。(同步了 offset 的部分数据，所以部分复制的基础就是偏移量 offset)

心跳：

> 主节点在建立成功后会维护这长连接彼此发送心跳检测

*   主从节点彼此都有心跳检测机制，各自模拟成对方的客户端进行通信，通过client list命令查看复制相关客户端信息，主节点的连接状态为flags=M,从节点连接状态 flags=S。
*   主节点默认每隔10秒对从节点发送ping命令，判断从节点的存活性和连接状态。可通过参数repl-ping-slave-period控制发送频率。
*   从节点在主线程中每隔1秒发送replconf ack {offset} 命令，给主节点上报自身当前的复制偏移量。

缓冲区大小调节：

*   由于缓冲区长度固定且有限，因此可以备份的写命令也有限，当主从节点offset的差距过大超过缓冲区长度时，将无法执行部分复制，只能执行全量复制。
*   反过来说，为了提高网络中断时部分复制执行的概率，可以根据需要增大复制积压缓冲区的大小(通过配置repl-backlog-size)来设置；
*   例如 如果网络中断的平均时间是 60s，而主节点平均每秒产生的写命令(特定协议格式)所占的字节数为100KB，则复制积压缓冲区的平均需求为6MB，保险起见， 可以设置为12MB，来保证绝大多数断线情况都可以使用部分复制。