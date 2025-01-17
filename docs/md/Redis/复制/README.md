# 复制

**概述**：主从复制可以在一定程度上扩展redis性能，redis的主从复制和关系型数据库的主从复制类似，从机能够精确的复制主机上的内		   容。实现了主从复制后，一方面能够实现数据的读写分离，降低master的压力，另一方面也能实现数据的备份。

**CAP 原理**  ：CAP 原理就好比分布式领域的牛顿定律，它是分布式存储的理论基石。  

- C - Consistent ，一致性
- A - Availability ， 可用性  
- P - Partition tolerance ， 分区容忍性  

分布式系统的节点往往都是分布在不同的机器上进行网络隔离开的，这意味着必然会有网络断开的风险，这个网络断开的场景的专业词汇叫着「 网络分区」  。

在网络分区发生时，两个分布式节点之间无法进行通信，我们对一个节点进行的修改操作将无法同步到另外一个节点，所以数据的「一致性」将无法满足，因为两个分布式节点的数据不再保持一致。除非我们牺牲「可用性」，也就是暂停分布式节点服务，在网络分区发时，不再提供修改数据的功能，直到网络状况完全恢复正常再继续对外提供服务。  

![image-20210522140453922](https://studyimages.oss-cn-beijing.aliyuncs.com/img/Redis/20220716084637.png)

**一句话概括 CAP 原理就是——网络分区发生时，一致性和可用性两难全。**  

[脑图](https://www.cnblogs.com/zjfjava/p/14255015.html)

![img](https://studyimages.oss-cn-beijing.aliyuncs.com/img/Redis/20220716084638.png)

## 1、配置  

### 1.1 建立复制  

参与复制的Redis实例划分为主节点（master） 和从节点（slave） 。 默认情况下， Redis都是主节点。 每个从节点只能有一个主节点， 而主节点可以同时具有多个从节点。 复制的数据流是单向的， 只能由主节点复制到从节点。 配置复制的方式有以下三种：  

1. 在配置文件中加入 **slaveof {masterHost} {masterPort}**随Redis启动生效 。如：slaveof 127.0.0.1 6379
2. 在redis-server启动命令后加入 **--slaveof {masterHost} {masterPort}生效。**
3. 直接使用命令： **slaveof {masterHost} {masterPort}生效**。  

综上所述， slaveof命令在使用时， 可以运行期动态配置， 也可以提前写到配置文件中。   例如本地启动两个端口为6379和6380的Redis节点， 在127.0.0.1： 6380执行如下命令：  

```
127.0.0.1:6380>slaveof 127.0.0.1 6379
```

slaveof配置都是在从节点发起， 这时6379作为主节点， 6380作为从节点。 复制关系建立后执行如下命令测试：  

```
127.0.0.1:6379>set hello redis
OK
127.0.0.1:6379>get hello
"redis"
127.0.0.1:6380>get hello
"redis"
```

针对主节点6379的任何修改都可以同步到从节点6380中，复制过程如图1-1所示。  

<img src="https://studyimages.oss-cn-beijing.aliyuncs.com/img/Redis/20220716084639.png" alt="image-20210521000840423" />

<div align="center">图1-1</div>

**slaveof**本身是异步命令，执行slaveof命令时，节点只保留主节点信息后返回，后续复制流程在节点内部异步执行。主从节点复制成功建立后，可以使用**info replication**来查看复制相关状态，如下图所示。

- 主节点6379复制状态信息：  

  ```
  127.0.0.1:6379>info replication
  # Replication
  role:master
  connected_slaves:1
  slave0:ip=127.0.0.1,port=6379,state=online,offset=43,lag=0
  ```

  

- 从节点6380复制状态信息：  

  ```
  127.0.0.1:6380>info replication
  # Replication
  role:slave
  master_host:127.0.0.1
  master_port:6380
  master_link_status:up
  master_last_io_seconds_ago:4
  master_sync_in_progress:0
  ...
  ```

  

### 1.2 断开复制

- slaveof命令不但可以建立复制， 还可以在从节点执行slaveof no one来断开与主节点复制关系。 例如在6380节点上执行**slaveof no one**来断开复制， 如图1-2所示 。

<img src="https://studyimages.oss-cn-beijing.aliyuncs.com/img/Redis/20220716084640.png" alt="image-20210521001929771" />

<div align="center">图1-2</div>

- 断开复制主要流程：
  1. 断开与主节点复制关系。
  2. 从节点晋升为主节点。  

- 从节点断开复制后并不会抛弃原有数据， 只是无法再获取主节点上的数据变化  ！

- 通过slaveof命令还可以实现切主操作， 所谓切主是指把当前从节点对主节点的复制切换到另一个主节点。 执行**slaveof {newMasterIp}  {newMasterPort}** 命令即可， 例如把6380节点从原来的复制6379节点变为复制6381节点， 如图1-3所示。 

  
  
  <img src="https://studyimages.oss-cn-beijing.aliyuncs.com/img/Redis/20220716084641.png" alt="image-20210521002632946" />
  
  <div align="center">图1-3</div>



切主操作流程如下：

- 断开与旧主节点复制关系。
- 与新主节点建立复制关系。
-  **删除从节点当前所有数据。**
- 对新主节点进行复制操作。  



### 1.3 安全性

对于数据比较重要的节点， 主节点会通过设置`requirepass`参数进行密码验证， 这时所有的客户端访问必须使用`auth`命令实行校验。 从节点与主节点的复制连接是通过一个特殊标识的客户端来完成， 因此需要配置从节点的`masterauth`参数与主节点密码保持一致， 这样从节点才可以正确地连接到主节点并发起复制流程。  



### 1.4 只读  

默认情况下， 从节点使用 **slave-read-only=yes** 配置为只读模式。 由于复制只能从主节点到从节点， 对于从节点的任何修改主节点都无法感知， 修改从节点会造成主从数据不一致。 **因此建议线上不要修改从节点的只读模式 。** 



### 1.6 传输延迟  

主从节点一般部署在不同机器上，复制时的网络延迟就成为需要考虑的问题，Redis为我们提供了repl-disable-tcp-nodelay参数用于控制是否关闭TCP_NODELAY，默认关闭，说明如下：

- 当关闭时，主节点产生的命令数据无论大小都会及时地发送给从节点，这样主从之间**延迟会变小**，但增加了网络带宽的消耗。适用于主从之间的网络环境良好的场景，如同机架或同机房部署。
- 当开启时，主节点会合并较小的TCP数据包从而省带宽。默认发送时间取决于Linux的内核，一般默认为40毫秒。这种配置节省了带宽但**增大主从之间的延迟**。适用于主从网络环境复杂或带宽紧张的场景，如跨机房部署。



***

## 2、复制原理

### 2.1 复制过程

从图2-1中可以看出复制过程大致分为6个过程：  

<img src="https://studyimages.oss-cn-beijing.aliyuncs.com/img/Redis/20220716084642.png" alt="image-20210523171750298" />

<div align="center">图2-1</div>

1. 保存主节点（master）信息

   执行slaveof后，从节点只保存主节点的地址信息便直接返回，这时建立复制流程还没有开始，在从节点6380执行info replication可以看到如下信息：

   ![image-20210523172408117](https://studyimages.oss-cn-beijing.aliyuncs.com/img/Redis/20220716084643.png)

   <div align="center">图2-2</div>

   从统计信息可以看出， 主节点的ip和port被保存下来， 但是主节点的连
   接状态（master_link_status） 是下线状态。 执行slaveof后Redis会打印如下日志：

   ![image-20210523172439534](https://studyimages.oss-cn-beijing.aliyuncs.com/img/Redis/20220716084644.png)  

   <div align="center">图2-3</div>

   

   

2. 从节点（slave）内部通过每秒运行的定时任务维护复制相关逻辑，当定时任务发现存在新的主节点后，会尝试与该节点建立网络连接，如图所示。

   ![image-20210523172651629](https://studyimages.oss-cn-beijing.aliyuncs.com/img/Redis/20220716084645.png)

   <div align="center">图2-4</div>

   从节点会建立一个socket套接字，例如图2-4从节点建立了一个端口为24555的套接字，专门用于接受主节点的复制命令。从节点连接成功后，打印如下日志：

   ![image-20210523173046367](https://studyimages.oss-cn-beijing.aliyuncs.com/img/Redis/20220716084646.png)

   如果从节点无法建立连接，定时任务会无限重试直到连接成功或者执行`slaveof no one`取消复制，如图2-5所示。

   <img src="https://studyimages.oss-cn-beijing.aliyuncs.com/img/Redis/20220716084647.png" alt="image-20210523173231452" />

   <div align="center">图2-5</div>

   

   关于连接失败， 可以在从节点执行info replication查看`master_link_down_since_seconds`指标， 它会记录与主节点连接失败的系统时
   间。 从节点连接主节点失败时也会每秒打印如下日志：  

   ![image-20210523173346394](https://studyimages.oss-cn-beijing.aliyuncs.com/img/Redis/20220716084648.png)

   

3. 发送ping命令  

   连接建立成功后从节点发送ping请求进行首次通信， ping请求主要目的如下：

   - 检测主从之间网络套接字是否可用。  
   - 检测主节点当前是否可接受处理命令。  
   - 如果发送ping命令后， 从节点没有收到主节点的pong回复或者超时， 比如网络超时或者主节点正在阻塞无法响应命令， 从节点会断开复制连接， 下次定时任务会发起重连，如图2-6所示。

   <img src="https://studyimages.oss-cn-beijing.aliyuncs.com/img/Redis/20220716084649.png" alt="image-20210523173607435" />

   <div align="center">图2-6</div>

   从节点发送的ping命令成功返回， Redis打印如下日志， 并继续后续复制流程：  

![image-20210523173642666](https://studyimages.oss-cn-beijing.aliyuncs.com/img/Redis/20220716084650.png)



4. 权限验证。 如果主节点设置了`requirepass`参数， 则需要密码验证，从节点必须配置`masterauth`参数保证与主节点相同的密码才能通过验证； 如果验证失败复制将终止， 从节点重新发起复制流程。  

5. 同步数据集。 主从复制连接正常通信后， 对于首次建立复制的场景， 主节点会把持有的数据全部发送给从节点， 这部分操作是耗时最长的步骤。 Redis在2.8版本以后采用新复制命令**psync**进行数据同步， 原来的**sync**命令依然支持， 保证新旧版本的兼容性。 新版同步划分两种情况： **全量同步**和**部分同步**。

6. 命令持续复制。 当主节点把当前的数据同步给从节点后， 便完成了复制的建立流程。 接下来主节点会持续地把写命令发送给从节点， 保证主从数据一致性。  



### 2.2 数据同步

Redis在2.8及以上版本使用**psync**命令完成主从数据同步， 同步过程分为： **全量复制**和**部分复制**。  

- 全量复制：一般用于初次复制场景，Redis早期支持的复制功能只有全量复制，他会把主节点全部数据一次性发送给从节点，当数据量较大时，会对主从节点和网络造成很大的开销。
- 部分复制：用于处理在主从复制中因网络闪断等原因造成的数据丢失场景，当从节点再次连上主节点后，如果条件允许，住系欸但会补发丢失数据给从节点。因为补发的数据远远小于全量数据，可以有效避免全量复制的过高开销。

部分复制是对老版复制的重大优化， 有效避免了不必要的全量复制操作。 因此当使用复制功能时， 尽量采用2.8以上版本的Redis。
psync命令运行需要以下组件支持：

- 主从节点各自复制偏移量。
- 主节点复制积压缓冲区。
- 主节点运行id。  

1. 复制偏移量  

    参与复制的主从节点都会维护自身复制偏移量。 主节点（master） 在处 理完写入命令后， 会把命令的字节长度做累加记录， 统计信息在info  relication中的 `master_repl_offset` 指标中：  

![image-20210523180042705](https://studyimages.oss-cn-beijing.aliyuncs.com/img/Redis/20220716084651.png)

​	从节点（slave） 每秒钟上报自身的复制偏移量给主节点， 因此主节点也会保存从节点的复制偏移量， 统计指标如下：  

![image-20210523180059887](https://studyimages.oss-cn-beijing.aliyuncs.com/img/Redis/20220716084652.png)

从节点在接收到主节点发送的命令后， 也会累加记录自身的偏移量。 统计信息在info relication中的 `slave_repl_offset` 指标中：  

![image-20210523180130347](https://studyimages.oss-cn-beijing.aliyuncs.com/img/Redis/20220716084653.png)

复制偏移量的维护如图2-7所示。  

![image-20210523180419505](https://studyimages.oss-cn-beijing.aliyuncs.com/img/Redis/20220716084654.png)

<div align="center">图2-7</div>

**通过对比主从节点的复制偏移量， 可以判断主从节点数据是否一致。**  



2. 复制积压缓冲区  

   复制积压缓冲区是保存在主节点上的一个固定长度的队列， 默认大小为1MB， 当主节点有连接的从节点（slave） 时被创建， 这时主节点（master）响应写命令时， 不但会把命令发送给从节点， 还会写入复制积压缓冲区， 如图2-8所示 。

   <img src="https://studyimages.oss-cn-beijing.aliyuncs.com/img/Redis/20220716084655.png" alt="image-20210523180408062" /> 

<div align="center">图2-8</div>

由于缓冲区本质上是先进先出的定长队列， 所以能实现保存最近已复制数据的功能， 用于部分复制和复制命令丢失的数据补救。 复制缓冲区相关统计信息保存在主节点的info replication中：  

<img src="https://studyimages.oss-cn-beijing.aliyuncs.com/img/Redis/20220716084656.png" alt="image-20210523180618269" />

根据统计指标， 可算出复制积压缓冲区内的可用偏移量范围：
[repl_backlog_first_byte_offset，repl_backlog_first_byte_offset+repl_backlog_histlen]。   



3. 主节点运行ID  

   每个Redis节点启动后都会动态分配一个40位的十六进制字符串作为运行ID。 运行ID的主要作用是用来唯一识别Redis节点， 比如从节点保存主节点的运行ID识别自己正在复制的是哪个主节点。 如果只使用 `ip+port` 的方式识别主节点， 那么主节点重启变更了整体数据集（如替换RDB/AOF文件） ，从节点再基于偏移量复制数据将是不安全的， 因此当运行ID变化后从节点将做全量复制。 可以运行info server命令查看当前节点的运行ID：  

![image-20210523180807171](https://studyimages.oss-cn-beijing.aliyuncs.com/img/Redis/20220716084657.png)

​	需要注意的是Redis关闭再启动后， 运行ID会随之改变， 例如执行如下命令  ：

​	![image-20210523180934370](https://studyimages.oss-cn-beijing.aliyuncs.com/img/Redis/20220716084658.png)

​	如何在不改变运行ID的情况下重启呢？当需要调优一些内存相关配置， 例如： hash-max-ziplist-value等， 这些配置需要Redis重新加	载才能优化已存在的数据， 这时可以使用debug reload命令重新加载RDB并保持运行ID不变， 从而有效避免不必要的全量复制。 命令	如下：  

![image-20210523181011109](https://studyimages.oss-cn-beijing.aliyuncs.com/img/Redis/20220716084659.png)



### 2.3 全量复制

全量复制是Redis主从第一次建立复制时必须经历的阶段。触发全量复制的命令是 `sync`和 `psync`，他们的对应版本如图所示。

<img src="https://studyimages.oss-cn-beijing.aliyuncs.com/img/Redis/20220716084700.png" alt="image-20210523192736727" />

这里主要介绍psync全量复制流程， 它与2.8以前的sync全量复制机制基本一致。 全量复制的完整运行流程如图所示。  

流程说明：  

1. 发送psync命令进行数据同步， 由于是第一次进行复制， 从节点没有复制偏移量和主节点的运行ID， 所以发送psync-1。  

2. 主节点根据psync-1解析出当前为全量复制， 回复+FULLRESYNC响应。  

3. 从节点接收主节点的响应数据保存运行ID和偏移量offset， 执行到当前步骤时从节点打印如下日志：  

   ![image-20210523193448757](https://studyimages.oss-cn-beijing.aliyuncs.com/img/Redis/20220716084701.png)

<img src="https://studyimages.oss-cn-beijing.aliyuncs.com/img/Redis/20220716084702.png" alt="image-20210523193723883" />

4. 主节点执行bgsave保存RDB文件到本地。

5. 主节点发送RDB文件给从节点， 从节点把接收的RDB文件保存在本地并直接作为从节点的数据文件， 接收完RDB后从节点打印相关日志， 可以在日志中查看主节点发送的数据量：

   ![image-20210523194345487](https://studyimages.oss-cn-beijing.aliyuncs.com/img/Redis/20220716084703.png)

6. 对于从节点开始接收RDB快照到接收完成期间， 主节点仍然响应读写命令， 因此主节点会把这期间写命令数据保存在复制客户端缓冲区内， 当从节点加载完RDB文件后， 主节点再把缓冲区内的数据发送给从节点， 保证主从之间数据一致性。 如果主节点创建和传输RDB的时间过长， 对于高流量写入场景非常容易造成主节点复制客户端缓冲区溢出。 默认配置为 `clientoutput-buffer-limit slave256MB64MB60`， 如果60秒内缓冲区消耗持续大于64MB或者直接超过256MB时， 主节点将直接关闭复制客户端连接， 造成全量同步失败。 对应日志如下：  

   ![image-20210523194728874](https://studyimages.oss-cn-beijing.aliyuncs.com/img/Redis/20220716084704.png)

7. 从节点接收完主节点传送来的全部数据后会清空自身旧数据， 该步骤对应如下日志：  

   ![image-20210523194807973](https://studyimages.oss-cn-beijing.aliyuncs.com/img/Redis/20220716084705.png)

8. 从节点清空数据后开始加载RDB文件， 对于较大的RDB文件， 这一步操作依然比较耗时， 可以通过计算日志之间的时间差来判断加载RDB的总耗时， 对应如下日志：  

   ![image-20210523194831341](https://studyimages.oss-cn-beijing.aliyuncs.com/img/Redis/20220716084706.png)

9. 从节点成功加载完RDB后， 如果当前节点开启了AOF持久化功能，它会立刻做bgrewriteaof操作， 为了保证全量复制后AOF持久化文件立刻可用。  



### 2.4 部分复制

部分复制主要是Redis针对全量复制的过高开销做出的一种优化措施，使用 `psync{runId}{offset}` 命令实现。 当从节点（slave） 正在复制主节点（master） 时， 如果出现**网络闪断**或者**命令丢失**等异常情况时， 从节点会向主节点要求补发丢失的命令数据， 如果主节点的复制积压缓冲区内存在这部分数据则直接发送给从节点， 这样就可以保持主从节点复制的一致性。 补发的这部分数据一般远远小于全量数据， 所以开销很小。 部分复制的流程如图所示 :

<img src="https://studyimages.oss-cn-beijing.aliyuncs.com/img/Redis/20220716084707.png" alt="image-20210523195235546" />

流程说明：  

1. 当主从节点之间网络出现中断时， 如果超过repl-timeout时间， 主节点会认为从节点故障并中断复制连接， 打印如下日志：  

   ![image-20210523195335001](https://studyimages.oss-cn-beijing.aliyuncs.com/img/Redis/20220716084708.png)

   如果此时从节点没有宕机， 也会打印与主节点连接丢失日志：  

   ![image-20210523195358111](https://studyimages.oss-cn-beijing.aliyuncs.com/img/Redis/20220716084709.png)

2. 主从连接中断期间主节点依然响应命令， 但因复制连接中断命令无法发送给从节点， 不过主节点内部存在的复制积压缓冲区， 依然可以保存最近一段时间的写命令数据， 默认最大缓存1MB。  

3. 当主从节点网络恢复后， 从节点会再次连上主节点， 打印如下日志：  

   ![image-20210523195446281](https://studyimages.oss-cn-beijing.aliyuncs.com/img/Redis/20220716084710.png)

4. 当主从连接恢复后， 由于从节点之前保存了自身已复制的偏移量和主节点的运行ID。 因此会把它们当作psync参数发送给主节点， 要求进行部分复制操作。 该行为对应从节点日志如下：  

   ![image-20210523195512601](https://studyimages.oss-cn-beijing.aliyuncs.com/img/Redis/20220716084711.png)

5. 主节点接到psync命令后首先核对参数runId是否与自身一致， 如果一致， 说明之前复制的是当前主节点； 之后根据参数offset在自身复制积压缓冲区查找， 如果偏移量之后的数据存在缓冲区中， 则对从节点发送+CONTINUE响应， 表示可以进行部分复制。 从节点接到回复后打印如下日志：    

   ![image-20210523195543793](https://studyimages.oss-cn-beijing.aliyuncs.com/img/Redis/20220716084712.png)

6. 主节点根据偏移量把复制积压缓冲区里的数据发送给从节点， 保证主从复制进入正常状态。 发送的数据量可以在主节点的日志获取， 如下所示：  

   ![image-20210523195600700](https://studyimages.oss-cn-beijing.aliyuncs.com/img/Redis/20220716084713.png)

   从日志中可以发现这次部分复制只同步了78字节， 传递的数据远远小于全量数据。  



### 2.6 心跳

主从节点在建立复制后， 它们之间维护着长连接并彼此发送心跳命令，如图所示。 

<img src="https://studyimages.oss-cn-beijing.aliyuncs.com/img/Redis/20220716084714.png" alt="image-20210523195714289" /> 

主从心跳判断机制：  

1. 主从节点彼此都有心跳检测机制， 各自模拟成对方的客户端进行通信， 通过client list命令查看复制相关客户端信息， 主节点的连接状态为flags=M， 从节点连接状态为flags=S。  
2. 主节点默认每隔10秒对从节点发送ping命令， 判断从节点的存活性和连接状态。 可通过参数repl-ping-slave-period控制发送频率。
3. 从节点在主线程中每隔1秒发送replconf ack{offset}命令， 给主节点上报自身当前的复制偏移量。 replconf命令主要作用如下：  
   - 实时监测主从节点网络状态。  
   - 上报自身复制偏移量， 检查复制数据是否丢失， 如果从节点数据丢失， 再从主节点的复制缓冲区中拉取丢失数据。  
   - 实现保证从节点的数量和延迟性功能， 通过min-slaves-to-write、 minslaves-max-lag参数配置定义。  

主节点根据replconf命令判断从节点超时时间， 体现在info replication统计中的lag信息中， lag表示与从节点最后一次通信延迟的秒数，正常延迟应该在0和1之间。 如果超过repl-timeout配置的值（默认60秒） ， 则判定从节点下线并断开复制客户端连接。 即使主节点判定从节点下线后， 如果从节点重新恢复， 心跳检测会继续进行。  



### 2.7 无盘复制  

所谓无盘复制是指主服务器直接通过套接字将快照内容发送到从节点，生成快照是一个遍历的过程，主节点会一边遍历内存，一遍将序
列化的内容发送到从节点，从节点还是跟之前一样，先将接收到的内容存储到磁盘文件中，再进行一次性加载。  



### 2.8 异步复制  

主节点不但负责数据读写， 还负责把写命令同步给从节点。 写命令的发送过程是异步完成， 也就是说主节点自身处理完写命令后直接返回给客户端， 并不等待从节点复制完成， 主节点复制流程如图所示。  

<img src="https://studyimages.oss-cn-beijing.aliyuncs.com/img/Redis/20220716084715.png" alt="image-20210523200128523" />

主节点复制流程：  

1. 主节点6379接收处理命令。
2. 命令处理完之后返回响应结果。
3. 对于修改命令异步发送给6380从节点， 从节点在主线程中执行复制的命令。  

由于主从复制过程是异步的， 就会造成从节点的数据相对主节点存在延迟。 具体延迟多少字节， 我们可以在主节点执行info replication命令查看相关指标获得。 如下：  

![image-20210523200243478](https://studyimages.oss-cn-beijing.aliyuncs.com/img/Redis/20220716084716.png)

在统计信息中可以看到从节点slave0信息， 分别记录了从节点的ip和port， 从节点的状态， offset表示当前从节点的复制偏移量，
`master_repl_offset`表示当前主节点的复制偏移量， 两者的差值就是当前从节点复制延迟量。 Redis的复制速度取决于主从之间网络环境，`repl-disabletcp-nodelay`， 命令处理速度等。 正常情况下， 延迟在1秒以内。  



***

## 3、基于复制的应用场景  

通过复制机制， 数据集可以存在多个副本（从节点） 。 这些副本可以应用于读写分离、 故障转移（failover） 、 实时备份等场景。   

### 3.1 读写分离  

对于读占比较高的场景， 可以通过把一部分读流量分摊到从节点（slave） 来减轻主节点（master） 压力， 同时需要注意永远只对主节点执行写操作， 如图所示。  

<img src="https://studyimages.oss-cn-beijing.aliyuncs.com/img/Redis/20220716084717.png" alt="image-20210523201225361" />

当使用从节点响应读请求时， 业务端可能会遇到如下问题：  

- 复制数据延迟。
- 读到过期数据。
- 从节点故障。  

1. 数据延迟  

   Redis复制数据的延迟由于异步复制特性是无法避免的， 延迟取决于网络带宽和命令阻塞情况， 比如刚在主节点写入数据后立刻在从节点上读取可能获取不到。 需要业务场景允许短时间内的数据延迟。 对于无法容忍大量延迟场景， 可以编写外部监控程序监听主从节点的复制偏移量， 当延迟较大时触发报警或者通知客户端避免读取延迟过高的从节点， 实现逻辑如图所示。  

   <img src="https://studyimages.oss-cn-beijing.aliyuncs.com/img/Redis/20220716084718.png" alt="image-20210523201358588" />

   说明如下：

   1） 监控程序（monitor） 定期检查主从节点的偏移量， 主节点偏移量在info replication的master_repl_offset指标记录， 从节点偏移	  量可以查询主节点的slave0字段的offset指标， 它们的差值就是主从节点延迟的字节量。  

   2） 当延迟字节量过高时， 比如超过10MB。 监控程序触发报警并通知客户端从节点延迟过高。 可以采用Zookeeper的监听回调机制	   实现客户端通知。
   3） 客户端接到具体的从节点高延迟通知后， 修改读命令路由到其他从节点或主节点上。 当延迟恢复后， 再次通知客户端， 恢复从	   节点的读命令请求。  

   

   这种方案的成本比较高， 需要单独修改适配Redis的客户端类库。 如果涉及多种语言成本将会扩大。 客户端逻辑需要识别出读写请求并自动路由，还需要维护故障和恢复的通知。 采用此方案视具体的业务而定， 如果允许不一致性或对延迟不敏感的业务可以忽略， 也可以采用Redis集群方案做水平扩展。  

2. 读到过期数据  

   当主节点存储大量设置超时的数据时， 如缓存数据， Redis内部需要维护过期数据删除策略， 删除策略主要有两种： 惰性删除和定时删除。

   **惰性删除：** 主节点每次处理读取命令时， 都会检查键是否超时， 如果超时则执行del命令删除键对象， 之后del命令也会异步发送给				   从节点。 需要注意的是为了保证复制的一致性， 从节点自身永远不会主动删除超时数据， 如图所示。

   <img src="https://studyimages.oss-cn-beijing.aliyuncs.com/img/Redis/20220716084719.png" alt="image-20210523201705874" />  

   

   **定时删除：** Redis主节点在内部定时任务会循环采样一定数量键，当发现采样的键过期时执行del命令，之后再同步给从节点，如图所示。  

<img src="https://studyimages.oss-cn-beijing.aliyuncs.com/img/Redis/20220716084720.png" alt="image-20210523201758625" />

如果此时数据大量超时， 主节点采样速度跟不上过期速度且主节点没有读取过期键的操作， 那么从节点将无法收到del命令。 这时在从节点上可以读取到已经超时的数据。 Redis在3.2版本解决了这个问题， 从节点读取数据之前会检查键的过期时间来决定是否返回数据， 可以升级到3.2版本来规避这个问题。   

3. 从节点故障问题   

   对于从节点的故障问题， 需要在客户端维护可用从节点列表， 当从节点故障时立刻切换到其他从节点或主节点上。 这个过程类似上文提到的针对延迟过高的监控处理， 需要开发人员改造客户端类库。  

   综上所出， 使用Redis做读写分离存在一定的成本。 建议大家在做读写分离之前， 可以考虑使用Redis Cluster等分布式解决方案，这样不止扩展了读性能还可以扩展写性能和可支撑数据规模， 并且一致性和故障转移也可以得到保证， 对于客户端的维护逻辑也相对容易 。

      

### 3.2 主从配置不一致  

对于有些配置主从之间是可以不一致， 比如： 主节点关闭AOF在从节点开启。 但对于内存相关的配置必须要一致， 比如`maxmemory`， `hash-max-ziplist-entries`等参数。 当配置的`maxmemory`从节点小于主节点， 如果复制的数据量超过从节点`maxmemory`时， 它会根据`maxmemory-policy`策略进行内存溢出控制， 此时从节点数据已经丢失， 但主从复制流程依然正常进行， 复制偏移量也正常。 修复这类问题也只能手动进行全量复制。 当压缩列表相关参数不一致时， 虽然主从节点存储的数据一致但实际内存占用情况差异会比较大。   

