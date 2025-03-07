---
title: Redis 集群
date: 2021-10-11
tags: 
 - Redis
categories: Redis
---

![](https://tva1.sinaimg.cn/large/008i3skNly1gv7y6sjivij60v60kumz402.jpg)



## 一、Redis 集群是啥

我们先回顾下前边介绍的几种 Redis 高可用方案：持久化、主从同步和哨兵机制。但这些方案仍有痛点，其中最主要的问题就是存储能力受单机限制，以及没办法实现写操作的负载均衡。

Redis 集群刚好解决了上述问题，实现了较为完善的高可用方案。



### 1.1 Redis 集群化

集群，即 Redis Cluster，是 Redis 3.0 开始引入的分布式存储方案。

集群由多个节点(Node)组成，Redis 的数据分布在这些节点中。集群中的节点分为主节点和从节点：只有主节点负责读写请求和集群信息的维护；从节点只进行主节点数据和状态信息的复制。



### 1.2 集群的主要作用

1. **数据分区**： 数据分区 *(或称数据分片)* 是集群最核心的功能。集群将数据分散到多个节点，**一方面** 突破了 Redis 单机内存大小的限制，**存储容量大大增加**；**另一方面** 每个主节点都可以对外提供读服务和写服务，**大大提高了集群的响应能力**。

   Redis 单机内存大小受限问题，例如，如果单机内存太大，`bgsave` 和 `bgrewriteaof` 的 `fork` 操作可能导致主进程阻塞，主从环境下主机切换时可能导致从节点长时间无法提供服务，全量复制阶段主节点的复制缓冲区可能溢出……

2. **高可用**： 集群支持主从复制和主节点的 **自动故障转移** *（与哨兵类似）*，当任一节点发生故障时，集群仍然可以对外提供服务。

![](https://cdn.jsdelivr.net/gh/Jstarfish/picBed/redis/redis-cluster-framework.png)

上图展示了 **Redis Cluster** 典型的架构图，集群中的每一个 Redis 节点都 **互相两两相连**，客户端任意 **直连** 到集群中的 **任意一台**，就可以对其他 Redis 节点进行 **读写** 的操作。



### 1.3 Redis 集群的基本原理

![](https://cdn.jsdelivr.net/gh/Jstarfish/picBed/redis/redis-cluster-slot.png)

Redis 集群中内置了 `16384` 个哈希槽。当客户端连接到 Redis 集群之后，会同时得到一份关于这个 **集群的配置信息**，当客户端具体对某一个 `key` 值进行操作时，会计算出它的一个 Hash 值，然后把结果对 `16384` **求余数**，这样每个 `key` 都会对应一个编号在 `0-16383` 之间的哈希槽，Redis 会根据节点数量 **大致均等** 的将哈希槽映射到不同的节点。

再结合集群的配置信息就能够知道这个 `key` 值应该存储在哪一个具体的 Redis 节点中，如果不属于自己管，那么就会使用一个特殊的 `MOVED` 命令来进行一个跳转，告诉客户端去连接这个节点以获取数据：

```bash
GET x
-MOVED 3999 127.0.0.1:6381
```

`MOVED` 指令第一个参数 `3999` 是 `key` 对应的槽位编号，后面是目标节点地址，`MOVED` 命令前面有一个减号，表示这是一个错误的消息。客户端在收到 `MOVED` 指令后，就立即纠正本地的 **槽位映射表**，那么下一次再访问 `key` 时就能够到正确的地方去获取了。



## 二、Hello World

#### 2.1 创建集群节点配置文件

创建六个配置文件，分别命名为：`redis_7000.conf`/`redis_7001.conf`…..`redis_7005.conf`，然后根据不同的端口号修改对应的端口值就好了（方便管理可以将这些配置文件放在同一个目录下，我这里放在了 `cluster_config` 目录下）：

```bash
# 后台执行
daemonize yes
# 端口号
port 7000
# 启动集群模式
cluster-enabled yes
# 每一个集群节点都有一个配置文件，这个文件是不能手动编辑的。确保每一个集群节点的配置文件不通
cluster-config-file nodes-7000.conf
# 集群节点的超时时间，单位：ms，超时后集群会认为该节点失败
cluster-node-timeout 5000
# 最后将 appendonly 改成 yes(AOF 持久化)
appendonly yes
```

#### 2.2 启动 Redis 实例

启动刚才配置的 6 个 Redis 实例

```bash
redis-server cluster_config/redis_7000.conf
redis-server cluster_config/redis_7001.conf
redis-server cluster_config/redis_7002.conf
redis-server cluster_config/redis_7003.conf
redis-server cluster_config/redis_7004.conf
redis-server cluster_config/redis_7005.conf 
```

然后执行 `ps -ef | grep redis` 查看是否启动成功：

![](https://cdn.jsdelivr.net/gh/Jstarfish/picBed/redis/redis-cluseter-ps.png)

可以看到 `6` 个 Redis 节点都以集群的方式成功启动了，**但是现在每个节点还处于独立的状态**，也就是说它们每一个都各自成了一个集群，还没有互相联系起来，我们需要手动地把他们之间建立起联系。

#### 2.3 建立集群

创建集群，其实就是节点执行下列命令（Redis 5 之后的方式，之前的版本可以使用 redis-trib.rb 创建）：

```bash
redis-cli --cluster create 127.0.0.1:7000 127.0.0.1:7001 127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005 --cluster-replicas 1
```

这里稍微解释一下这个 `--replicas 1` 的意思是：我们希望为集群中的每个主节点创建一个从节点。

观察控制台输出：

![](https://cdn.jsdelivr.net/gh/Jstarfish/picBed/redis/redis-cluster-new.jpg)

看到 `[OK]` 的信息之后，就表示集群已经搭建成功了，可以看到，这里我们正确地创建了三主三从的集群。

（这里可能会遇到一些坑，槽没有被完全覆盖，或者 node 不为空这种错误）

#### 2.4 验证集群

我们先使用 `redic-cli` 任意连接一个节点：

```bash
redis-cli -c -h 127.0.0.1 -p 7000
127.0.0.1:7000>
```

`-c` 表示集群模式；`-h` 指定 ip 地址；`-p` 指定端口。

然后随便 `set` 一些值观察控制台输入：

```bash
127.0.0.1:7000> set name javakeeper
-> Redirected to slot [5798] located at 127.0.0.1:7001
OK
127.0.0.1:7001> 
```

可以看到这里 Redis 自动帮我们进行了 `Redirected` 操作跳转到了 `7001` 这个实例上。

我们再使用 `cluster info` *(查看集群信息)* 和 `cluster nodes` *(查看节点列表)* 来分别看看：*(任意节点输入均可)*

![](https://cdn.jsdelivr.net/gh/Jstarfish/picBed/redis/cluster-info.png)



## 三、深入集群原理

Redis 集群最核心的功能就是数据分区，数据分区之后又伴随着通信机制和数据结构的建设，所以我们从这 3 个方面来一一深入

### 3.1 数据分区方案

数据分区有**顺序分区**、**哈希分区**等，其中哈希分区由于其天然的随机性，使用广泛；集群的分区方案便是哈希分区的一种。

哈希分区的基本思路是：对数据的特征值（如key）进行哈希，然后根据哈希值决定数据落在哪个节点。常见的哈希分区包括：<mark>哈希取余分区、一致性哈希分区、带虚拟节点的一致性哈希分区等</mark>。

#### 方案一：哈希取余分区

哈希取余分区思路非常简单：计算 `key` 的 hash 值，然后对节点数量进行取余，从而决定数据映射到哪个节点上。

不过该方案最大的问题是，**当新增或删减节点时**，节点数量发生变化，系统中所有的数据都需要 **重新计算映射关系**，引发大规模数据迁移。

这种方式的突出优点是简单性，常用于数据库的分库分表规则，一般采用预分区的方式，提前根据数据量规划好分区数，比如划分为 512  或 1024 张表，保证可支撑未来一段时间的数据量，再根据负载情况将表迁移到其他数据库中。扩容时通常采用翻倍扩容，避免数据映射全部被打乱导致全量迁移的情况

#### 方案二：一致性哈希分区

一致性哈希算法将 **整个哈希值空间** 组织成一个虚拟的圆环，范围一般是 0 - $2^{32}$，对于每一个数据，根据 `key` 计算 hash 值，确定数据在环上的位置，然后从此位置沿顺时针行走，找到的第一台服务器就是其应该映射到的服务器：

![](https://cdn.jsdelivr.net/gh/Jstarfish/picBed/redis/redis-consistency.png)

与哈希取余分区相比，一致性哈希分区将 **增减节点的影响限制在相邻节点**。以上图为例，如果在 `node1` 和 `node2` 之间增加 `node5`，则只有 `node2` 中的一部分数据会迁移到 `node5`；如果去掉 `node2`，则原 `node2` 中的数据只会迁移到 `node3` 中，只有 `node3` 会受影响。

一致性哈希分区的主要问题在于，当 **节点数量较少** 时，增加或删减节点，**对单个节点的影响可能很大**，造成数据的严重不平衡。还是以上图为例，如果去掉 `node2`，`node3` 中的数据由总数据的 `1/4` 左右变为 `1/2` 左右，与其他节点相比负载过高。

#### 方案三：带有虚拟节点的一致性哈希分区

该方案在 **一致性哈希分区的基础上**，引入了 **虚拟节点** 的概念。Redis 集群使用的便是该方案，其中的虚拟节点称为 **槽（slot）**。槽是介于数据和实际节点之间的虚拟概念，每个实际节点包含一定数量的槽，每个槽包含哈希值在一定范围内的数据。槽的范围一般远大于节点数。

在使用了槽的一致性哈希分区中，**槽是数据管理和迁移的基本单位**。槽 **解耦** 了 **数据和实际节点** 之间的关系，增加或删除节点对系统的影响很小。仍以上图为例，系统中有 `4` 个实际节点，假设为其分配 `16` 个槽(0-15)；

- 槽 0-3 位于 node1；4-7 位于 node2；以此类推….

如果此时删除 `node2`，只需要将槽 4-7 重新分配即可，例如槽 4-5 分配给 `node1`，槽 6 分配给 `node3`，槽 7 分配给 `node4`；可以看出删除 `node2` 后，数据在其他节点的分布仍然较为均衡。



Redis 虚拟槽分区的特点:

- 解耦数据和节点之间的关系，简化了节点扩容和收缩难度。 

- 节点自身维护槽的映射关系，不需要客户端或者代理服务维护槽分区元数据。

- 支持节点、槽、键之间的映射查询，用于数据路由、在线伸缩等场景。



### 3.2 集群功能限制

Redis 集群相对单机在功能上存在一些限制，需要开发人员提前了解，在使用时做好规避。限制如下:

- key 批量操作支持有限。如 mset、mget，目前只支持具有相同 slot 值的 key 执行批量操作。对于映射为不同 slot 值的 key 由于执行 mget、mget 等操作可能存在于多个节点上因此不被支持。

  (为此，Redis 引入 HashTag 的概念，使得数据分布算法可以根据 key 的某一部分进行计算，让相关的两条记录落到同一个数据分片，**当一个key包含 {} 的时候，就不对整个 key 做 hash，而仅对 {} 包括的字符串做 hash**。Pipeline 同样可以受益于 hash_tag)

  ```bash
  127.0.0.1:7000> mset javaframework Spring cframework Libevent
  (error) CROSSSLOT Keys in request don't hash to the same slot
  127.0.0.1:7000> mset java{framework} Spring c{framework} Libevent
  -> Redirected to slot [10840] located at 127.0.0.1:7001
  OK
  127.0.0.1:7001> mget java{framework} c{framework}
  1) "Spring"
  2) "Libevent"
  127.0.0.1:7001> 
  ```

- key 事务操作支持有限。同理只支持多 key 在同一节点上的事务操作，当多个 key 分布在不同的节点上时无法使用事务功能。

- key 作为数据分区的最小粒度，因此不能将一个大的键值对象如 hash、list 等映射到不同的节点。

- 不支持多数据库空间。单机下的 Redis 可以支持 16 个数据库，集群模式下只能使用一个数据库空间，即 db0。

- 复制结构只支持一层，从节点只能复制主节点，不支持嵌套树状复制结构。



### 3.3 节点通信

集群的建立离不开节点之间的通信，例如我们上面启动六个集群节点之后通过 `redis-cli` 命令帮助我们搭建起来了集群，实际上背后每个集群之间的两两连接是通过了 `CLUSTER MEET  ` 命令发送 `MEET` 消息完成的。

通信过程说明：

1. 集群中的每个节点都会单独开辟一个 TCP 通道，用于节点之间彼此通信，通信端口号在基础端口上加 10000
2. 每个节点在固定周期内通过特定规则选择几个节点发送 ping 消息
3. 接收到 ping 消息的节点用 pong 消息作为响应

集群中每个节点通过一定规则挑选要通信的节点，每个节点可能知道全部节点，也可能仅知道部分节点，只要这些节点彼此可以正常通信，最终它们会达到一致的状态。当节点出故障、新节点加入、主从角色变化、槽信息 变更等事件发生时，通过不断的 `ping/pong` 消息通信，经过一段时间后所有的节点都会知道整个集群全部节点的最新状态，从而达到集群状态同步的目的。

#### 两个端口

在 **哨兵系统** 中，节点分为 **数据节点** 和 **哨兵节点**：前者存储数据，后者实现额外的控制功能。在 **集群** 中，没有数据节点与非数据节点之分：**所有的节点都存储数据，也都参与集群状态的维护**。为此，集群中的每个节点，都提供了两个 TCP 端口：

- **普通端口：** 即我们在前面指定的端口 *(7000等)*。普通端口主要用于为客户端提供服务 *（与单机节点类似）*；但在节点间数据迁移时也会使用。
- **集群端口：** 端口号是普通端口 + 10000 *（10000是固定值，无法改变）*，如 `7000` 节点的集群端口为 `17000`。**集群端口只用于节点之间的通信**，如搭建集群、增减节点、故障转移等操作时节点间的通信；不要使用客户端连接集群接口。为了保证集群可以正常工作，在配置防火墙时，要同时开启普通端口和集群端口。



#### Gossip 协议

> 对于一个分布式集群来说，它的良好运行离不开集群节点信息和节点状态的正常维护。为了实现这一目标，通常我们可以选择**中心化**的方法，使用一个第三方系统，比如 Zookeeper 或 etcd，来维护集群节点的信息、状态等。同时，我们也可以选择**去中心化**的方法，让每个节点都维护彼此的信息、状态，并且使用集群通信协议 Gossip 在节点间传播更新的信息，从而实现每个节点都能拥有一致的信息。下图就展示了这两种集群节点信息维护的方法，你可以看下。
>
> ![](https://cdn.jsdelivr.net/gh/Jstarfish/picBed/redis/redis-gossip.png)

节点间通信，按照通信协议可以分为几种类型：单对单、广播、Gossip 协议等。重点是广播和 Gossip 的对比。

- 广播是指向集群内所有节点发送消息。**优点** 是集群的收敛速度快(集群收敛是指集群内所有节点获得的集群信息是一致的)，**缺点** 是每条消息都要发送给所有节点，CPU、带宽等消耗较大。

- Gossip 协议的特点是：在节点数量有限的网络中，**每个节点都 “随机” 的与部分节点通信** *（并不是真正的随机，而是根据特定的规则选择通信的节点），经过一番杂乱无章的通信，每个节点的状态很快会达到一致。Gossip 协议的 **优点** 有负载 (比广播)* 低、去中心化、容错性高 *(因为通信有冗余)* 等；**缺点** 主要是集群的收敛速度慢。

  （为什么需要随机呢？ ）

  Gossip 协议工作原理就是节点彼此不断通信交换信息，一段时间后所有的节点都会知道集群完整的信息，这种方式类似流言传播。

Gossip 协议的主要职责就是信息交换。信息交换的载体就是节点彼此发送的 Gossip 消息，了解这些消息有助于我们理解集群如何完成信息交换。

#### 消息类型

集群中的节点采用 **固定频率（每秒10次）** 的 **定时任务** 进行通信相关的工作：判断是否需要发送消息及消息类型、确定接收节点、发送消息等。如果集群状态发生了变化，如增减节点、槽状态变更，通过节点间的通信，所有节点会很快得知整个集群的状态，使集群收敛。

节点间发送的消息主要分为 `5` 种：`meet 消息`、`ping 消息`、`pong 消息`、`fail 消息`、`publish 消息`。不同的消息类型，通信协议、发送的频率和时机、接收节点的选择等是不同的：

![](https://cdn.jsdelivr.net/gh/Jstarfish/picBed/redis/redis-message-type.png)

- **MEET 消息：** 用于通知新节点加入。消息发送者通知接收者加入到当前集群，meet 消息通信正常完成后，接收节点会加入到集群中并进行周期性的 ping、pong 消息交换。
- **PING 消息：** 集群里每个节点每秒钟会选择部分节点发送 `PING` 消息，接收者收到消息后会回复一个 `PONG` 消息。**PING 消息的内容是自身节点和部分其他节点的状态信息**，作用是彼此交换信息，以及检测节点是否在线。`PING` 消息使用 Gossip 协议发送，接收节点的选择兼顾了收敛速度和带宽成本（内部频繁进行信息交换，而且 ping/pong 消息会携带当前节点和部分其他节点的状态数据，势必会加重带宽和计算的负担，所以选择需要通信的节点列表就很重要了），**具体规则如下**：
  1. 随机找 5 个节点，在其中选择最久没有通信的 1 个节点；
  2. 扫描节点列表，选择最近一次收到 `PONG` 消息时间大于 `cluster_node_timeout / 2` 的所有节点，防止这些节点长时间未更新。

![](https://cdn.jsdelivr.net/gh/Jstarfish/picBed/redis/redis-cluster-ping.png)

- **PONG消息：** `PONG` 消息封装了自身状态数据。可以分为两种：
  1. **第一种** 是在接到 `MEET/PING` 消息后回复的 `PONG` 消息；
  2. **第二种** 是指节点向集群广播 `PONG` 消息，这样其他节点可以获知该节点的最新信息，例如故障恢复后新的主节点会广播 `PONG` 消息。
- **FAIL 消息：** 当节点判定集群内另一个节点下线时，会向集群内广播一个 fail 消息，其他节点接收到 fail 消息之后把对应节点更新为下线状态。
- **PUBLISH 消息：** 节点收到 `PUBLISH` 命令后，会先执行该命令，然后向集群广播这一消息，接收节点也会执行该 `PUBLISH` 命令。



#### 消息结构

所有的消息格式划分为：**消息头**和**消息体**。消息头包含发送节点自身状态数据，接收节点根据消息头就可以获取到发送节点的相关数据，结构如下（ `src/cluster.h` 目录下可以大概看下源码）:

```c
typedef struct {
    char sig[4];        /* 信号标示 */
    uint32_t totlen;    /* 消息总长度 */
    uint16_t ver;       /* 协议版本 */
    uint16_t port;      /* TCP base port number. */
    uint16_t type;      /* 消息类型 */
    uint16_t count;     /* Only used for some kind of messages. */
    uint64_t currentEpoch;  /* 当前发送节点的配置纪元 */
    uint64_t configEpoch;   /* 主节点/从节点的主节点配置纪元 */
    uint64_t offset;    /* 复制偏移量 */
    char sender[CLUSTER_NAMELEN]; /* Name of the sender node */
    unsigned char myslots[CLUSTER_SLOTS/8];
    char slaveof[CLUSTER_NAMELEN];
    char myip[NET_IP_STR_LEN];    /* Sender IP, if not all zeroed. */
    char notused1[34];  /* 34 bytes reserved for future usage. */
    uint16_t cport;      /* Sender TCP cluster bus port */
    uint16_t flags;      /* 发送节点标识,区分主从角色，是否下线等 */
    unsigned char state; /* 发送节点所处的集群状态 */
    unsigned char mflags[3]; /* 消息标识: CLUSTERMSG_FLAG[012]_... */
    union clusterMsgData data; /* 消息正文 */
} clusterMsg;
```

集群内所有的消息都采用相同的**消息头**结构 `clusterMsg`，它包含了发送节点关键信息，如节点 id、槽映射、节点标识(主从角色，是否下线)等。

**消息体** 在 Redis 内部采用 `clusterMsgData` 结构声明，结构如下:

```c
union clusterMsgData {
    //Ping、Pong和Meet消息类型对应的数据结构
    struct {
        /* Array of N clusterMsgDataGossip structures */
        clusterMsgDataGossip gossip[1];
    } ping;

    //Fail消息类型对应的数据结构
    struct {
        clusterMsgDataFail about;
    } fail;

    //Publish消息类型对应的数据结构
    struct {
        clusterMsgDataPublish msg;
    } publish;

    //Update消息类型对应的数据结构
    struct {
        clusterMsgDataUpdate nodecfg;
    } update;

    //Module消息类型对应的数据结构
    struct {
        clusterMsgModule msg;
    } module;
};
```

消息体 `clusterMsgData` 定义发送消息的数据，其中 ping、meet、pong 都采用 `clusterMsgDataGossip` 数组作为消息体数据，实际消息类型使用消息头的 type 属性区分。每个消息体包含该节点的多个 `clusterMsgDataGossip` 结构数据，用于信息交换，结构如下:

```c
typedef struct {
    char nodename[CLUSTER_NAMELEN]; //节点名称
    uint32_t ping_sent;  //节点发送Ping的时间
    uint32_t pong_received; //节点收到Pong的时间
    char ip[NET_IP_STR_LEN];  //节点IP
    uint16_t port;              //节点和客户端的通信端口
    uint16_t cport;             //节点用于集群通信的端口
    uint16_t flags;             //节点的标记
    uint32_t notused1;    //未用字段
} clusterMsgDataGossip;
```

从 clusterMsgDataGossip 数据结构中，我们可以看到，它里面包含了节点的基本信息，比如节点名称、IP 和通信端口，以及使用 Ping、Pong 消息发送和接收时间来表示的节点运行状态。

消息交互的过程就是解析消息头和消息体的过程

- 解析消息头过程：消息头包含了发送节点的信息，如果发送节点是新节点且消息是 meet 类型，则加入到本地节点列表；如果是已知节点，则尝试更新发送节点的状态，如槽映射关系、主从角色等状态。
- 解析消息体过程：如果消息体的 `clusterMsgDataGossip` 数组包含的节点是新节点，则尝试发起与新节点的 meet 握手流程；如果是已知节点，则根据 `clusterMsgDataGossip` 中的 flags 字段判断该节点是否下线，用于故障转移

消息处理完后回复 pong 消息，内容同样包含消息头和消息体，发送节点接收到回复的 pong 消息后，采用类似的流程解析处理消息并更新与接收节点最后通信时间，完成一次消息通信。



#### 数据结构

节点需要专门的数据结构来存储集群的状态。所谓集群的状态，是一个比较大的概念，包括：集群是否处于上线状态、集群中有哪些节点、节点是否可达、节点的主从状态、槽的分布……

节点为了存储集群状态而提供的数据结构中，最关键的是 `clusterNode` 和 `clusterState` 结构：前者记录了一个节点的状态，后者记录了集群作为一个整体的状态。

`clusterNode` 结构保存了 **一个节点的当前状态**，包括创建时间、节点 id、ip 和端口号等。每个节点都会用一个 `clusterNode` 结构记录自己的状态，并为集群内所有其他节点都创建一个 `clusterNode` 结构来记录节点状态。

下面列举了 `clusterNode` 的部分字段，并说明了字段的含义和作用（`src/cluster.h`）：

```c
typedef struct clusterNode {
    //节点创建时间
    mstime_t ctime;
    //节点id
    char name[REDIS_CLUSTER_NAMELEN];
    //节点的ip和端口号
    char ip[REDIS_IP_STR_LEN];
    int port;
    //节点标识：整型，每个bit都代表了不同状态，如节点的主从状态、是否在线、是否在握手等
    int flags;
    //配置纪元：故障转移时起作用，类似于哨兵的配置纪元
    uint64_t configEpoch;
    //槽在该节点中的分布：占用16384/8个字节，16384个比特；每个比特对应一个槽：比特值为1，则该比特对应的槽在节点中；比特值为0，则该比特对应的槽不在节点中
    unsigned char slots[16384/8];
    //节点中槽的数量
    int numslots;
    …………
} clusterNode;
```

除了上述字段，`clusterNode` 还包含节点连接、主从复制、故障发现和转移需要的信息等。

`clusterState` 结构保存了在当前节点视角下，集群所处的状态。主要字段包括（`src/cluster.h`）：

```c
typedef struct clusterState {
    clusterNode *myself;  //自身节点
    uint64_t currentEpoch;	   //配置纪元
    //集群状态：在线还是下线
    int state;
    //集群中至少包含一个槽的节点数量
    int size;
    //哈希表，节点名称->clusterNode节点指针
    dict *nodes;
    //槽分布信息：数组的每个元素都是一个指向clusterNode结构的指针；如果槽还没有分配给任何节点，则为NULL
    clusterNode *slots[16384];
    …………
} clusterState;
```

除此之外，`clusterState` 还包括故障转移、槽迁移等需要的信息。



### 3.4 集群自动故障转移

Redis 集群自身实现了高可用。高可用首先需要解决集群部分失败的场景：当集群内少量节点出现故障时通过自动故障转移保证集群可以正常对外提供服务。

之前我们了解了哨兵机制的故障发现和故障转移。集群的实现有些思路类似。

#### 3.4.1 故障发现

通过定时任务发送 PING 消息检测其他节点状态；节点下线分为**主观下线**和**客观下线**；

- 主观下线：集群中每个节点都会定期向其他节点发送 ping 消息，接收节点回复 pong 消息作为响应。如果在 `cluster-node-timeout` 时间内通信一直失败，则发送节点会认为接收节点存在故障，把接收节点标记为主观下线(pfail)状态

- 当某个节点判断另一个节点主观下线后，相应的节点状态会跟随消息在集群内传播。ping/pong 消息的消息体会携带集群 1/10 的其他节点状态数据， 当接受节点发现消息体中含有主观下线的节点状态时，会在本地找到故障节点的 ClusterNode 结构，保存到下线报告链表中。结构如下:

  ```c
  struct clusterNode {    /* 认为是主观下线的clusterNode结构 */ 
    list *fail_reports; /* 记录了所有其他节点对该节点的下线报告 */ ...
  };
  ```

  通过 Gossip 消息传播，集群内节点不断收集到故障节点的下线报告。当半数以上持有槽的主节点都标记某个节点是主观下线时。触发客观下线流程。

  这里有两个问题:

  1. 为什么必须是负责槽的主节点参与故障发现决策?

     因为集群模式下 只有处理槽的主节点才负责读写请求和集群槽等关键信息维护，而从节点只进行主节点数据和状态信息的复制。

  2. 为什么半数以上处理槽的主节点?

     必须半数以上是为了应对网络分区等原因造成的集群分割情况，被分割的小集群因为无法完成从主观下线到 客观下线这一关键过程，从而防止小集群完成故障转移之后继续对外提供服务。

#### 3.4.2 故障恢复

故障节点变为客观下线后，如果下线节点是持有槽的主节点则需要在它的从节点中选出一个替换它，从而保证集群的高可用。

下线主节点的所有从节点承担故障恢复的义务，当从节点通过内部定时任务发现自身复制的主节点进入客观下线时，将会触发故障恢复流程：

1. 资格检查

   每个从节点都要检查最后与主节点断线时间，判断是否有资格替换故障的主节点。如果从节点与主节点断线时间超过 `cluster-node-time*cluster-slave-validity-factor`，则当前从节点不具备故障转移资格。参数 `cluster-slave- validity-factor` 用于从节点的有效因子，默认为 10。

2. 准备选举时间

   当从节点符合故障转移资格后，更新触发故障选举的时间，只有到达该时间后才能执行后续流程

   ```c
   struct clusterState { 
     ...
   	mstime_t failover_auth_time; /* 记录之前或者下次将要执行故障选举时间 */
   	int failover_auth_rank; /* 记录当前从节点排名 */ }
   ```

   这里之所以采用延迟触发机制，主要是通过对多个从节点使用不同的延迟选举时间来支持优先级问题。复制偏移量越大说明从节点延迟越低，那么它应该具有更高的优先级来替换故障主节点。

3. 发起选举

   当从节点定时任务检测到达故障选举时间(`failover_auth_time`)到达后，发起选举流程如下:

   - 更新配置纪元

     配置纪元是一个只增不减的整数，每个主节点自身维护一个配置纪元 (`clusterNode.configEpoch`)标示当前主节点的版本，所有主节点的配置纪元 都不相等，从节点会复制主节点的配置纪元。整个集群又维护一个全局的配 置纪元(`clusterState.current Epoch`)，用于记录集群内所有主节点配置纪元的最大版本。

   - 广播选举消息

4. 选举投票

   只有持有槽的主节点才会处理故障选举消息 (`FAILOVER_AUTH_REQUEST`)，因为每个持有槽的节点在一个配置纪元内都有唯一的一张选票，当接到第一个请求投票的从节点消息时回复 `FAILOVER_AUTH_ACK` 消息作为投票，之后相同配置纪元内其他从节点的选举消息将忽略。

   投票过程其实是一个领导者选举的过程，如集群内有 N 个持有槽的主节点代表有 N 张选票。由于在每个配置纪元内持有槽的主节点只能投票给一个 从节点，因此只能有一个从节点获得N/2+1的选票，保证能够找出唯一的从节点。

   Redis 集群没有直接使用从节点进行领导者选举，主要因为从节点数必须大于等于 3 个才能保证凑够 N/2+1 个节点，将导致从节点资源浪费。使用集群内所有持有槽的主节点进行领导者选举，即使只有一个从节点也可以完成选举过程。

   当从节点收集到 N/2+1 个持有槽的主节点投票时，从节点可以执行替换主节点操作，例如集群内有 5 个持有槽的主节点，主节点 b 故障后还有 4 个， 当其中一个从节点收集到 3 张投票时代表获得了足够的选票可以进行替换主节点操作。

​		![](https://cdn.jsdelivr.net/gh/Jstarfish/picBed/redis/redis-cluster-vote.png)

5. 替换主节点

   当从节点收集到足够的选票之后，触发替换主节点操作:

   - 当前从节点取消复制变为主节点。

   - 执行 clusterDelSlot 操作撤销故障主节点负责的槽，并执行 clusterAddSlot 把这些槽委派给自己。

   - 向集群广播自己的 pong 消息，通知集群内所有的节点当前从节点变为主节点并接管了故障主节点的槽信息。



与哨兵一样，集群只实现了主节点的故障转移；从节点故障时只会被下线，不会进行故障转移。因此，使用集群时，应谨慎使用读写分离技术，因为从节点故障会导致读服务不可用，可用性变差。



### 3.5 客户端访问集群

#### 3.5.1 redis-cli

当节点收到 redis-cli 发来的命令(如 set/get )时，过程如下：

1. 计算 key 属于哪个槽：`CRC16(key) & 16383`

   集群提供的 cluster keyslot 命令也是使用上述公式实现

   ```bash
   127.0.0.1:7001> cluster keyslot k1
   (integer) 12706
   ```

2. 判断 key 所在的槽是否在当前节点

   假设 key 位于第 i 个槽，`clusterState.slots[i]` 则指向了槽所在的节点，如果 `clusterState.slots[i]==clusterState.myself`，说明槽在当前节点，可以直接在当前节点执行命令；否则，说明槽不在当前节点，则查询槽所在节点的地址(`clusterState.slots[i].ip/port`)，并将其包装到 MOVED 错误中返回给 redis-cli。

3. redis-cli 收到 MOVED 错误后，根据返回的 ip 和 port 重新发送请求

像 redis-cli 这种客户端又叫 Dummy(傀儡)客户端，它优点是代码实现简单，对客户端协议影响较小，只需要根据重定向信息再次发送请求即可。但是它的弊端很明显，每次执行键命令前都要到 Redis 上进行重定向才能找到要执行命令的节点，额外增加了 IO 开销， 这不是Redis 集群高效的使用方式。正因为如此通常集群客户端都采用另一 种实现：Smart(智能)客户端。

#### 3.5.2 Smart客户端

大多数开发语言的 Redis 客户端都采用 Smart 客户端支持集群协议。Smart 客户端通过在内部维护 slot→node 的映射关系，本地就可实现键到节点的查找，从而保证 IO 效率的最大化，而 MOVED 重定向负责协助 Smart 客户端更新 slot→node 映射。

以 Jedis 为例，说明 Smart 客户端操作集 群的流程：

1. 首先在 JedisCluster 初始化时会选择一个运行节点，初始化槽和节点映射关系，使用 `cluster slots` 命令完成

   ```bash
   127.0.0.1:7001> cluster slots
   1) 1) (integer) 10923   // 开始槽范围
      2) (integer) 16383		// 结束槽范围
      3) 1) "127.0.0.1"    //主节点ip
         2) (integer) 7002	//从节点端口
         3) "911f517c6cc3501d42b9bba4aa58b5633375abb5"
      4) 1) "127.0.0.1"
         2) (integer) 7004
         3) "897b5abb7f79117bb645d79671ced5bebcb855ad"
   2) 1) (integer) 5461
      2) (integer) 10922
      3) 1) "127.0.0.1"
         2) (integer) 7001
         3) "f130468fc299509d709b0867b111342576f00b23"
      4) 1) "127.0.0.1"
         2) (integer) 7003
         3) "cc122bc7aa0d36da44a5d8c7fbe061940aa81a1d"
   127.0.0.1:7001> 
   ```

2. JedisCluster 解析 cluster slots 结果缓存在本地，并为每个节点创建唯一的 JedisPool 连接池。映射关系在 `JedisClusterInfoCache `类中
3. 当执行命令时，JedisCluster 根据 key->slot->node 选择需要连接的节点，发送命令。如果成功，则命令执行完毕。如果执行失败，则会随机选择其他节点进行重试，并在出现 MOVED 错误时，使用 cluster slots 重新同步 slot->node 的映射关系



### 参考与来源

1. https://redis.io/topics/cluster-tutorial
2. 《Redis 设计与实现》
3. 《Redis 开发与运维》
4. https://www.cnblogs.com/kismetv/p/9853040.html