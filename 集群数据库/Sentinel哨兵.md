# Sentinel哨兵

英 [ˈsentɪnl] 

## 总结

- Sentinel 只是一个运行在特殊模式下的 Redis 服务器，它使用了和普通模式不同的命令表，所以 Sentinel 模式能够使用的命令和普通 Redis 服务器能够使用的命令不同。

- Sentinel 会读人用户指定的配置文件，为每个要被监视的主服务器创建相应的实例结构，并创建连向主服务器的命令连接和订阅连接，其中命令连接用于向主服务器发送命令请求，而订阅连接则用于接收指定频道的消息。

- Sentinel 通过向主服务器发送INFO 命令来获得主服务器属下所有从服务器的地址信息，并为这些从服务器创建相应的实例结构，以及连向这些从服务器的命令连接和订阅连接。

- 在一般情况下，Sentinel 以每十秒一次的频率向被监视的主服务器和从服务器发送INFO 命令,当主服务器处于下线状态，或者 Sentinel 正在对主服务器进行故障转移操作时，Sentinel 向从服务器发送 INFO 命令的频率会改为每秒一次

- 对于监视同一个主服务器和从服务器的多个 Sentinel 来说，它们会以每两秒一次的频率，通过向被监视服务器的 `__sentinel__:hello`频道发送消息来向其他 Sentinel 宣告自己的存在。

- 每个 Sentinel 也会从 `__sentinel__:hello` 频道中接收其他 Sentinel 发来的信息，并根据这些信息为其他 Sentinel 创建相应的实例结构，以及命令连接。
- Sentinel 只会与主服务器和从服务器创建命令连接和订阅连接，Sentinel 与 Sentinel之间则只创建命令连接。

-  Sentinel 以每秒一次的频率向实例（包括主服务器、从服务器、其他 Sentinel）发送 PING 命令，并根据实例对 PNG 命令的回复来判断实例是否在线，当一个实例在指定的时长中连续向 Sentinel 发送无效回复时，Sentinel 会将这个实例判断为主观下线。
- 当 Sentinel将一个主服务器判断为主观下线时，它会向同样监视这个主服务器的其他Sentinel 进行询问，看它们是否同意这个主服务器已经进人主观下线状态。

- 当 Sentinel 收集到足够多的主观下线投票之后，它会将主服务器判断为客观下线，并发起一次针对主服务器的故障转移操作。

## 介绍

![image-20190819135915785](assets/Sentinel哨兵/image-20190819135915785.png)

Sentinel(哨岗、哨兵)是 Redis的高可用性(high availability)解决方案：由一个或多个Sentinel 实例(Instance)组成的Sentinel 系统可以监视任意多个主服务器，Sentinel 系统以及这些主服务器属下的所有从服务器，并在被监视的主服务器进入下线状态时，自动将下线主服务器属下的某个从服务器升级为新的主服务器，然后由新的主服务器代替已下线的主服务器继续处理命令请求。

**故障处理过程**

当 server1 的下线时长超过用户设定的下线时长上限时，Sentinel 系统就会对 server1 执行故障转移操作

- 首先，Sentinel 系统会挑选 server1 属下的其中一个从服务器，并将这个被选中的从服务器升级为新的主服务器。

- 之后，Sentinel 系统会向 server1 属下的所有从服务器发送新的复制指令，让它们成为新的主服务器的从服务器，当所有从服务器都开始复制新的主服务器时，故障转移操作执行完毕。

- 另外，Sentinel 还会继续监视已下线的 server1, 并在它重新上线时，将它设置为新的主服务器的从服务器

![image-20190819140024916](assets/Sentinel哨兵/image-20190819140024916.png)

将原来的主服务器变成从服务器.

![image-20190819140055295](assets/Sentinel哨兵/image-20190819140055295.png)



## Sentinel启动和初始化

Sentinel本质上是一个运行在特殊模式下的 Redis 服务器

```shell
$ redis-sentinel /path/to/your/sentinel.conf
```

当一个 Sentinel 启动时，它需要执行以下步骤：

1) 初始化服务器。

2) 将普通 Redis 服务器使用的代码替换成 Sentinel 专用代码。

​    端口和支持的命令都不一样.26379,PING、SENTINEL、INFO、SUBSCRIBE、UNSUBSCRIBE、PSUBSCRIBE 和 PUNSUBSCRIBE.

3) 初始化 Sentinel 状态

4) 根据给定的配置文件，初始化 Sentinel 的监视主服务器列表。

5) 创建连向主服务器的命令连接和订阅连接。



### 初始化服务器

首先，**因为 Sentinel 本质上只是一个运行在特殊模式下的 Redis 服务器**，所以启动 Sentinel  的第一步，就是初始化一个普通的 Redis 服务器.

不过，因为 Sentinel 执行的工作和普通 Redis 服务器执行的工作不同，所以 Sentinel 的初始化过程和普通Redis 服务器的初始化过程并不完全相同

例如，普通服务器在初始化时会通过载入 RDB 文件或者 AOF 文件来还原数据库状态，但是因为 Sentinel 并不使用数据库，所以初始化 Sentinel 时就不会载入 RDB 文件或者 AOF 文件。

| 功能                                           | 使用情况                                                     |
| ---------------------------------------------- | ------------------------------------------------------------ |
| 数据库和健值对方面的命令，比如 SET,DEL,FLUSHDB | 不使用                                                       |
| 事务命令，比如 MULT 和 WATCH                   | 不使用                                                       |
| 脚本命令，比如 EWAL                            | 不使用                                                       |
| RDB 持久化命令，比如 SAVE 和 BGSAVE            | 不使用                                                       |
| AOF 持久化命令，比如 BGREWRITEAOF              | 不使用                                                       |
| 复制命令，比如 SLAVEOF                         | Sentinel 内部可以使用，但客户端不可以使用                    |
| 发布与订阅命令，比如 PUBLISH ,SUBSCRIBE        | SUBSCRIBE、PSUBSCRIBE、UNSUBSCRIBE、PUNSUBSCRIBE 四个命令在 Sentinel 内部和客户端都可以使用，但 PUBLISH 命今只能在 Sentinel 内部使用 |
| 文件事件处理器（负责发送命令请求、处理命令回复 | Sentinel 内部使用，但关联的文件事件处理器和普通 Redis 服务器不同 |
| 时间事件处理器（负责执行 serverCron 函数）     | Sentinel 内部使用，时间事件的处理器仍然是 serverCron 函数，serverCron 函数会调用 `sentinel.c/sentinel1Timer` 函数，后者包含了 Sentinel 要执行的所有操作 |



### 使用Sentinel 专用代码

启动Sentinel的第二个步骤就是将一部分普通 Redis 服务器使用的代码替换成 Sentinel 专用代码。比如说，普通 Redis 服务器使用` redis.c/ REDIS_SERVER_PORT`常量的值作为服务器端口

```c
#define REDIS_SERVERPORT 6379
```

而 Sentinel 则使用 `sentinel.c/ REDIS_SENTINEL_PORT`常量的值作为服务器端口

```
 #define REDIS_SENTINEL_PORT 26379
```

除此之外，普通 Redis 服务器使用`redis.c/ redisCommandTable 作为服务器的命令表

```c
struct redisCommand redisCommandTable[] = {
  {"get", getcommand, 2, "r", 0, NULL,1,1.1.0, 0},
  {"set", setcommand, -3, "wm",0, nopreloadgetkeys, 1,1 1,0,0},
  {"setnx", setnxcommand, 3, "wm",0, nopreloadgetkeys, 1, 1, 1.0, 0},
   .....
}
```

而 Sentinel 则使用 `sentinel.c/sentinelcmds `作为服务器的命令表,并且其中的`INFO`命令会使用 Sentinel 模式下的专用实现 `sentinel1.c/ sentinelInfoCommand` 函数,而不是普通 Redis 服务器使用的实现` redis. c/infoCommand `函数：

```c
struct redisCommand sentinelcmds [] = {

 {"ping", pingCommand, 1, "",0, NULL,0, 0,0,0,0},
  {"sentinel", sentinelCommand, -2, "", O, NULL,0, 0,0,0,0},
  {"subscribe", subscribeCommand, 2, "", O, NULL,0, 0,0,0,0},
  {"unsubscribe", unsubscribeCommand, -1, "", O, NULL, 0,0,0,0, 0},
  {"psubscribe", psubscribeCommand, -2,"",0, NULL,, 0,0,0,0}
  {"punsubscribe", punsubscribeCommand, -1, "", O, NULL,0, 0,0,0,0},
  {"info", sentinelinfoCommand, -1, "",0, NULL,0,0,0, 0,0}
};
```

sentinelcmds 命令表也解释了为什么在 Sentinel 模式下,Redis 服务器不能执行诸如 SET、DBSIZE、EVAL 等等这些命令，因为服务器根本没有在命令表中载入这些命令。**PING、SENTINEL、INFO、SUBSCRIBE、UNSUBSCRIBE、PSUBSCRIBE 和 PUNSUBSCRIBE 这七个命令就是客户端可以对 Sentinel 执行的全部命令了。**

### 初始化 Sentinel 状态

在应用了Sentinel 的专用代码之后，接下来，服务器会初始化一个`sentinel.c/sentinelState` 结构（后面简称“Sentinel 状态”），这个结构保存了服务器中所有和 Sentinel 功能有关的状态（服务器的一般状态仍然由 `redis.h/redisServer` 结构保存）:

```c
Struct sentinelState{

//当前纪元，用于实现故障转移
uint64_t current_epoch;

//保存了所有被这个 sentinel 监视的主服务器 
//字典的健是主服务器的名字
//字典的值则是一个指向 sentinelRedisInstance 结构的指针 
dict *masters;

//是否进入了titl模式
int titl;

//目前正在执行的脚本的数量
int running_scripts;

//进入titl模式的时何
mistime_t tilt_start_time;

//最后一次执行时间处理器的时间
mistime_t previous_time:

//ー个 FTFO 队列，包含了所有需要执行的用户脚本
list scripts_queue;
}
```



### 初始化 Sentinel 状态的 masters 属性

Sentinel 状态中的 masters 字典记录了所有被 Sentinel 监视的主服务器的相关信息，其中

- 字典的键是被监视主服务器的名字。
- 字典的值则是被监视主服务器对应的`sentinel.c/ sentinelRedisInstance` 结构。

每个` sentinelRedisInstance` 结构(后面简称“实例结构”)代表一个被 Sentinel 监视的 Redis 服务器实例,这个实例可以是主服务器、从服务器，或者另外一个Sentinel.

实例结构包含的属性非常多，实例结构在表示主服务器时使用的部分主要有:

```c
typedef struct sentinelRedisInstance {
    // 标识值，记录了实例的类型，以及该实例的当前状态
    int flags;

    // 实例的名字
    // 主服务器的名字由用户在配置文件中设置
    // 从服务器以及 Sentinel 的名字由 Sentinel 自动设置
    // 格式为 ip:port ，例如 "127.0.0.1:26379"
    char *name;

    // 实例的运行 ID
    char *runid;

    // 配置纪元，用于实现故障转移
    uint64_t config_epoch;

    // 实例的地址  IP + PORT
    sentinelAddr *addr;

    // SENTINEL down-after-milliseconds 选项设定的值
    // 实例无响应多少毫秒之后才会被判断为主观下线（subjectively down）
    mstime_t down_after_period;

    // SENTINEL monitor <master-name> <IP> <port> <quorum> 选项中的 quorum 参数
    // 判断这个实例为客观下线（objectively down）所需的支持投票数量
    int quorum;

    // SENTINEL parallel-syncs <master-name> <number> 选项的值
    // 在执行故障转移操作时，可以同时对新的主服务器进行同步的从服务器数量 比如1
    int parallel_syncs;

    // SENTINEL failover-timeout <master-name> <ms> 选项的值
    // 刷新故障迁移状态的最大时限
    mstime_t failover_timeout;

    // ...

} sentinelRedisInstance;
```

对 Sentinel 状态的初始化将引发对 `masters` 字典的初始化， 而 `masters` 字典的初始化是根据被载入的 Sentinel 配置文件来进行的。

举个例子， 如果用户在启动 Sentinel 时， 指定了包含以下内容的配置文件：

```shell
#####################
# master1 configure #
#####################

sentinel monitor master1 127.0.0.1 6379 2

sentinel down-after-milliseconds master1 30000

sentinel parallel-syncs master1 1

sentinel failover-timeout master1 900000

#####################
# master2 configure #
#####################

sentinel monitor master2 127.0.0.1 12345 5

sentinel down-after-milliseconds master2 50000

sentinel parallel-syncs master2 5

sentinel failover-timeout master2 450000
```

那么 Sentinel 将为主服务器 `master1` 创建如图 IMAGE_MASTER1 所示的实例结构， 并为主服务器 `master2` 创建如图 IMAGE_MASTER2 所示的实例结构， 而这两个实例结构又会被保存到 Sentinel 状态的 `masters` 字典中.

![image-20190930182407906](assets/Sentinel哨兵/image-20190930182407906.png)



![image-20190930182425164](assets/Sentinel哨兵/image-20190930182425164.png)

![image-20190930182437721](assets/Sentinel哨兵/image-20190930182437721.png)



### 创建连向主服务器的两个连接

初始化 Sentinel 的最后一步是创建连向被监视主服务器的网络连接，**Sentinel 将成为主服务器的客户端**，它可以向主服务器发送命令，并从命令回复中获取相关的信息。

对于每个被 Sentinel 监视的主服务器来说，Sentinel 会创建两个连向主服务器的异步网络连接

![image-20190819201022837](assets/Sentinel哨兵/image-20190819201022837.png)

- 一个是命令连接，这个连接专门用于向主服务器发送命令，并接收命令回复。
- 另一个是订阅连接，这个连接专门用于订阅主服务器的 sentinel: hello 频道。

**为什么有两个连接?**

在 Redis 目前的发布与订阅功能中，被发送的信息都不会保存在 Redis 服务器里面

**如果在信息发送时，想要接收信息的客户端不在或者断线，那么这个客户端就会丢失这条信息**。因此，为了不丢失 sentinel: hello 频道的任何信息，Sentinel 必须专门用一个订阅连接来接收诚频道的信息。

另一方面，除了订阅频道之外，Sentinel 还必须向主服务器发送命令，以此来与主服务器进行通信，所以 Sentinel 还必须向主服务器创建命令连接。

因为 Sentinel 需要与多个实例创建多个网络连接，所以 Sentinel 使用的是异步连接。

> 异步连接,不必等待上一个消息的结果.

## 获取主服务器信息

Sentinel 默认会以每十秒一次的频率，通过命令连接向被监视的主服务器发送` INFO `命令，并通过分析 INFO 命令的回复来获取主服务器的当前信息。

![image-20190930193440522](assets/Sentinel哨兵/image-20190930193440522.png)

获得回复

```
# Server
run_id:dfasdfjasdjkfljasdflk

# Replication 
role: master

slave0: ip=127.0.0.1, port=11111, state=online, offset-43, lag=0
slave1: ip=127.0.0.1, port=22222, state=online, offset-43, lag=0
slave2: ip=127.0.0.1, port=33333, state=online, offset-43, lag=0
# Other sections
```

通过分析主服务器的 INFO 命令回复，Sentinel 可以获取以下两方面的信息：

- 一方面是关于主服务器本身的信息，包括 runid 域记录的服务器运行ID，以及role 域记录的服务器角色

- 另一方面是关于主服务器属下所有从服务器的信息，每个从服务器都由一个slave 字符串开头的行记录，每行的 ip=域记录了从服务器的ip 地址，而 port=域则记录了从服务器的端口号。

  根据这些 ip 地址和端口号，Sentinel 无须用户提供从服务器的地址信息，就可以自动发现从服务器。



根据 runid 域和 role 域记录的信息，Sentinel 将对主服务器的实例结构进行更新，例如，主服务器重启之后，它的运行 ID 就会和实例结构之前保存的运行 ID 不同，Sentinel 检测到这一情况之后，就会对实例结构的运行 ID 进行更新。

至于主服务器返回的从服务器信息，则会被用于更新主服务器实例结构的 slaves 字典，这个字典记录了主服务器属下从服务器的名单：

- 字典的键是由 Sentinel 自动设置的从服务器名字，格式为 ip: port：如对于 IP 地址为 127.0.0,1, 端口号为 11111 的从服务器来说，Sentinel 为它设置的名字就是 127.0.0.1:11111。

- 至于字典的值则是从服务器对应的实例结构：比如说，如果键是 127.0.0.1:11111

  那么这个键的值就是 P 地址为 127.0.0,1, 端口号为 1111 的从服务器的实例结构。

Sentinel 在分析 INFO 命令中包含的从服务器信息时，会检查从服务器对应的实例结构是否已经存在于 slaves 字典

- 如果从服务器对应的实例结构已经存在，那么 Sentinel对从服务器的实例结构进行更新
- 如果从服务器对应的实例结构不存在，那么说明这个从服务器是新发现的从服务器，Sentinel 会在 slaves 字典中为这个从服务器新创建一个实例结构。



更新结构

![image-20190819204045754](assets/Sentinel哨兵/image-20190819204045754.png)



## 获取从服务器信息

**当 Sentinel 发现主服务器有新的从服务器出现时(注意这是从上一步的主服务器INFO命令反馈发现的)，**Sentinel 除了会为这个新的从服务器创建相应的实例结构之外，Sentinel还会创建连接到从服务器的命令连接和订阅连接。

![image-20190930194121696](assets/Sentinel哨兵/image-20190930194121696.png)





在创建命令连接之后，Sentinel 在默认情况下，会以每十秒一次的频率通过命令连接向从服务器发送 INFO 命令，并获得类似于以下内容的回复：

```
# Server
run_id:dfasdfjasdjkfljasdflk

# Replication 
role: slave
master_host:127.0.0.1 
master_port: 6379
master_link_status: up

slave_repl_offset: 11887
slave_priority:100
# Other sections
```

根据 INFO 命令的回复，Sentinel 会提取出以下信息：

- 从服务器的运行 ID run id。
- 从服务器的角色 role。

- 主服务器的 IP 地址 master host，以及主服务器的端口号 master port 
- 主从服务器的连接状态 master_link_status。
- 从服务器的优先级 slave_ priority
-  从服务器的复制偏移量 slave_repl_offset。

根据这些信息，Sentinel 会对从服务器的实例结构进行更新.

![image-20190930194229373](assets/Sentinel哨兵/image-20190930194229373.png)



## 向主服务器,从服务器发送消息

在默认情况下，Sentinel会以每两秒一次的频率，通过命令连接向所有被监视的主服务器和从服务器发送以下格式的命令.**这些命令是当前Sentinel记录在sentinelState的信息**

```
PUBLISH sentinel: hello " <s_ip>, <s_port>, <s_runid>, <s_epoch>, <m_name>, <m_ip>,  <m _port>, <m_epoch>
```

这条命令向服务器的 sentinel: hello 频道发送了一条信息,信息的内容由多个参数组成:

- 其中以 s 开头的参数记录的是 Sentinel 本身的信息

- 而 m 开头的参数记录的则是主服务器的信息

  如果 Sentinel 正在监视的是主服务器，那么这些参数记录的就是主服务器的信息；

  如果 Sentinel 正在监视的是从服务器，那么这些参数记录的就是从服务器正在复制的主服务器的信息。

| 参数    | 意义                                          |
| ------- | --------------------------------------------- |
| s_ip    | Sentinel 的 IP 地址                           |
| s_port  | Sentinel 的端口号                             |
| s_runid | Sentinel 的运行 ID                            |
| s_epoch | Sentinel 当前的配置纪元（configuration epoch) |



| 参数    | 意义                   |
| ------- | ---------------------- |
| m_name  | 主服务名字             |
| m_ip    | 主服务器IP地址         |
| m_port  | 主服务器端口号         |
| m_epoch | 主服务器当前的配置纪元 |



以下是一条 Sentinel 通过 PUBLISH 命令向主服务器发送的信息示例：

```
127.0.0.1,26379, e955b4c85598ef5b5f055bc7ebfd5e828bed4fa,0, mymaster,127.0,0,1,6379,0
```

这个示例包含了以下信息：

- Sentinel 的 IP 地址为 127.0.0,1 端口号为 26379, 运行ID 为e955b4c85598ef5b5f055bc7ebfd5e828bed4fa，当前的配置纪元为0

- 主服务器的名字为 mymaster, IP 地址为 127.0,0,1, 端口号为 6379, 当前的配置纪元为0.

## 接收来自主服务器和从服务器的频道信息

当 Sentinel 与ー个主服务器或者从服务器建立起订阅连接之后，Sentinel 就会通过订阅连接，向服务器发送以下命令`SUBSCRIBE __sentinel__:hello`

Sentinel对`__sentinel__:hello`频道的订阅会一直持续到 Sentinel 与服务器的连接断开为止。

这也就是说，对于每个与 Sentinel连接的服务器，Sentinel 既通过命令连接向服务器的`__sentinel__:hello`频道发送信息，又通过订阅连接从服务器的 `__sentinel__:hello`频道接收信息.

![image-20190930195430510](assets/Sentinel哨兵/image-20190930195430510.png)



举个例子，假设现在有 sentinel、sentinel2、sentinel3 三个 Sentinel 在监视同一个服务器，那么当 sentinel1 向服务器的`__sentinel__:hello`频道发送一条信息时，所有订阅了 `__sentinel__:hello`频道的 Sentinel（包括 sentinel1自己在内）都会收到这条信息。

![image-20190930195543812](assets/Sentinel哨兵/image-20190930195543812.png)

当一个 Sentinel 从`__sentinel__:hello`频道收到一条信息时，Sentinel 会对这条信息进行分析,提取出信息中的 Sentinel IP 地址、Sentinel 端口号、Sentinel 运行 ID 等八个参数，并进行以下检查:

- 如果信息中记录的 Sentinel 运行 ID 和接收信息的 Sentinel 的运行 ID 相同，那么说明这条信息是 Sentinel 自己发送的，Sentinel 将丢弃这条信息，不做进一步处理。
- 相反地，如果信息中记录的 Sentinel 运行 ID 和接收信息的 Sentinel 的运行 ID 不相同，那么说明这条信息是监视同一个服务器的其他 Sentinel 发来的，接收信息的 Sentinel将根据信息中的各个参数，对相应主服务器的实例结构进行更新。

### **更新 sentinels 字典**

Sentinel 为主服务器创建的实例结构中的 sentinels 字典保存了除 Sentinel本身之外,所有同样监视这个主服务器的其他 Sentinel 的资料

- sentinels 字典的键是其中一个Sentinel 的名字，格式为 ip: port，比如对于IP 地址为 127.0,0.1, 端口号为 26379 的 Sentinel 来说，这个 Sentinel 在sentinels 字典中的键就是 127.0.0.1:26379
- sentinels 字典的值则是健所对应 Sentinel的实例结构，比如对于键"127.0.01:26379"来说，这个键在 sentinels 字典中的值就是IP 为 127.0,0.1, 端口号为 26379 的 Sentinel 的实例结构。

当一个 Sentinel接收到其他 Sentinel 发来的信息时（我们称呼发送信息的 Sentinel 为源 Sentinel，接收信息的 Sentinel 为目标 Sentinel），目标 Sentinel 会从信息中分析并提取出以下两方面参数：

- 与 Sentinel 有关的参数：源 Sentinel 的 P 地址、端口号、运行 ID 和配置纪元。

- 与主服务器有关的参数：源 Sentinel正在监视的主服务器的名字、IP 地址、端口号和配置纪元

根据信息中提取出的主服务器参数，目标 Sentinel 会在自己的 Sentinel 状态的 masters 字典中查找相应的主服务器实例结构，然后根据提取出的Sentinel 参数，检查主服务器实例结构的 sentinels 字典中，源 Sentinel 的实例结构是否存在：

- 如果源 Sentinel 的实例结构已经存在，那么对源 Sentinel 的实例结构进行更新。

- 如果源 Sentinel 的实例结构不存在，那么说明源 Sentinel是刚刚开始监视主服务器的新 Sentinel

  目标 Sentinel 会为源 Sentinel 创建一个新的实例结构，并将这个结构添加到 sentinels 字典里面。



举个例子，假设分别有 127.0,0.1:26379、127.0.0.1:26380、127.0.0.1:26381

三个 Sentinel 正在监视主服务器 127.0.0.1:6379, 那么当 127.0.0.1:26379 这个 Sentinel 接收到以下信息时

```
1)"message"
2)"__sentinel__:hello""
3)"127.0.0.1,26379,e955b4c85598ef5b5f055bc7efd5e828dhed4fa,0,mymaster,127,0,0.1,6379,0


1)"message"
2)"__sentinel__:hello"
3)"127.0.0.1,26381,6241bf5cf9bfc8ecd5d6eb6cc3185edfbb24903,0,mymaster,127.0.0.1,6379,0


1)"message"
2)"__sentinel__:hello"
3)"127.0.0.1,26380,a9b22fb79ae8fad28e4ea77d20398f776b89377,0,mymaster,127.0.0.1,6379,0
```

Sentinel 将执行以下动作

- 第一条信息的发送者为 127.0.0.1:26379 自己，这条信息会被忽略。
- 第二条信息的发送者为 127.0.0.1:26381, Sentinel会根据这条信息中提取出的内容，对sentinels字典中 127.0,0.1:26381对应的实例结构进行更新。
- 第三条信息的发送者为127.0.0.1:26380, Sentinel 会根据这条信息中提取出的内容，对 sentinels字典中 127.0.0.1:26380 所对应的实例结构进行更新。

![image-20190930201211851](assets/Sentinel哨兵/image-20190930201211851.png)

因为一个 Sentinel 可以通过分析接收到的频道信息来获知其他 Sentinel 的存在，并通过发送频道信息来让其他 Sentinel 知道自己的存在，所以用户在使用 Sentinel 的时候并不需要提供各个 Sentinel 的地址信息，**监视同一个主服务器的多个 Sentinel 可以自动发现对方**.



### 创建连向其他 Sentinel 的命令连接

当 Sentinel 通过频道信息发现一个新的 Sentinel 时，它不仅会为新Sentinel 在sentinels 字典中创建相应的实例结构，还会创建一个连向新 Sentinel 的命令连接，而新 Sentinel 也同样会创建连向这个Sentinel 的命令连接，最终监视同一主服务器的多个 Sentinel将形成相互连接的网络：Sentinel A 有连向 Sentinel B 的命令连接，而 Sentinel B 也有连向 Sentinel A 的命令连接。

![image-20190930201353770](assets/Sentinel哨兵/image-20190930201353770.png)

使用命令连接相连的各个 Sentinel 可以通过向其他 Sentinel 发送命令请求来进行信息交换,**主观下线检测和客观下线检测都会使用 Sentinel 之间的命令连接来进行通信。**

Sentinel 之间不会创建订阅连接

> Sentinel 在连接主服务器或者从服务器时，会同时创建命令连接和订阅逹接，但是在连接其他 Sentinel 时，却只会创建命令连接，而不创建订阅连接。这是因为 Sentinel 需要通过接收主服务器或者从服务器发来的频道信息来发現未知的新 Sentinel，所以オ需要建立订闻连接，而相互已知的 Sentinel 只要使用命令接来进行通信就足够了。

## 检测主观下线状态

在默认情况下，**Sentinel 会以每秒一次的频率向所有与它创建了命令连接的实例(包括主服务器、从服务器、其他 Sentinel 在内)发送 PING 命令**，并通过实例返回的 PING 命令回复来判断实例是否在线.

- 有效回复：实例返回+PONG、- LOADING、MASTERDOWN 三种回复的其中一种。
- 无效回复：实例返回除+PONG、- LOADING、MASTERDOWN 三种回复之外的其他回复，或者在指定时限内没有返回任何回复。

Sentinel 配置文件中的 `down-after-millseconds` 选项指定了 Sentinel 判断实例进入主观下线所需的时间长度：如果一个实例在 `down-after-millseconds` 毫秒内,连续向 Sentinel:返回无效回复，那么 Sentinel 会修改这个实例所对应的实例结构，在结构的 f1ags 属性中打开 SRI_S_DOWN 标识,以此来表示这个实例已经进人主观下线状态。



**主观下线时长选项的作用范围**

用户设置的`down-after-millseconds` 选项的值，不仅会被**本Sentinel** 用来判断主服务器的主观下线状态，还会被用于判断主服务器属下的所有从服务器，以及所有同样监视这个主服务器的其他Sentinel 的主观下线状态。举个例子，如果用户向 Sentinel设置了以下配置

```
sentinel monitor master 127.0.0.1 6379 2
sentinel down-after-milliseconds master 50000
```

那么 50000 毫秒不仅会成为 Sentinel 判断 master 进入主观下线的标准，还会成为 Sentinel 判断 master 属下所有从服务器，以及所有同样监视 master 的其他 Sentinel 进入主观下线的标准。

**多个 Sentinel 设置的主观下线时长可能不同**

`down-after-millseconds`选项另一个需要注意的地方是，对于监视同一个主服务器的多个 Sentinel 来说，这些 Sentinel 所设置的 `down-after-millseconds` 选项的值也可能不同，因此当一个 Sentinel。将主服务器判断为主观下线时，其他 Sentinel 可能仍然会认为主服务器处于在线状态。举个例子，知果 Sentinel 载入了以下配置：

```
sentinel monitor master 127.0.0.1 6379 2
sentinel down-after-milliseconds master 50000
```

而 Sentinel2 则载入了以下配置

```
sentinel monitor master 127.0.0.1 6379 2
sentinel down-after-milliseconds master 10000
```

那么当 master 的断线时长超过 10000 毫秒之后，Sentinel2 会将 master 判断为主观下线，而 Sentinel 却认为 master 仍然在线。只有当 master 的断线时长超过 50000 毫秒之后，Sentinel1和 Sentinel2オ会都认为 master 进入了主观下线状态。



## 检测客观下线状态

当 Sentinel将一个主服务器判断为主观下线之后，为了确认这个主服务器是否真的下线了，它会向同样监视这一主服务器的其他 Sentinel 进行询问，看它们是否也认为主服务器已经进入了下线状态（可以是主观下线或者客观下线）。当 Sentinel 从其他 Sentinel 那里接收到足够数量的已下线判断之后，Sentinel 就会将从服务器判定为客观下线，并对主服务器执行故障转移操作。

### 发送 SENTINEL is-master-down-by-adrr 命令

Sentinel 使用

`SENTINEL is-master-down-by-addr  <ip>  <port>  <current epoch> <runid>`

命令询问其他 Sentinel 是否同意主服务器已下线.

| 参数          | 意义                                                         |
| ------------- | ------------------------------------------------------------ |
| ip            | 被 Sentinel 判断为主观下线的主服务器的IP 地址                |
| port          | 被 Sentinel 判断为主观下线的主服务器的端口号                 |
| current_epoch | Sentinel 当前的配置纪元，用于选举领头 Sentinel               |
| Ruined        | 可以是*符号或者 Sentinel 的运行ID：*符号代表命令仅仅用于检测主服务器的客观下线状态，而 Sentinel 的运行ID 则用于选举领头 Sentinel |



举个例子，如果被 Sentinel 判断为主观下线的主服务器的 IP 为 127.0.0.1, 端口号为

6379, 并且 Sentinel 当前的配置纪元为0, 那么 Sentinel 将向其他 Sentinel 发送以下命令

`SENTINEL is-master-down-by-addr 127.0.0.1 6379 0`

### 接收 SENTINEL is-master-down-by-addr 命令

当一个 Sentinel(目标 Sentinel)接收到另一个 Sentinel(源 Sentinel)发来的 `SENTINEL is-master-down-by `命令时，目标 Sentinel 会分析并取出命令请求中包含的各个参数，并根据其中的主服务器 IP 和端口号，检查主服务器是否已下线，然后向源 Sentinel 返回条包含三个参数的 Multi Bulk 回复作为 `SENTINEL is-master-down-addr` 命令的回复：

| 参数         | 意义                                                         |
| ------------ | ------------------------------------------------------------ |
| down_state   | 返回目标 Sentinel 对主服务器的检查结果，1 代表主服务器已下线，0 代表主服务器未下线 |
| leader_runid | 可以是*符号或者目标 Sentinel的局部领头Sentinel 的运行 ID:     *符号代表命令仅仅用于检测主服务器的下线状态，而局部领头 Sentinel 的运行ID 则用于选举领头 Sentinel |
| leader_epoch | 目标 Sentinel的局部领头Sentinel的配置纪元,仅在 leader_runid 的值不为时有效，如果 1eader_runid的值为*，那么 leader_ epoch 总为 0 |

举个例子，如果一个 Sentinel 返回以下回复作为 `SENTINEL is-master-down-by addr` 命令的回复

1)  1
2) *
3) 0

那么说明 Sentinel 也同意主服务器已下线。



### 接收 SENTINEL is-master-down-by-addr 命令的回复

根据其他 Sentinel发回的`SENTINEL_is-master-down-by-addr `命令回复，Sentinel 将统计其他 Sentinel 同意主服务器已下线的数量，当这一数量达到配置指定的判断客观下线所需的数量时，Sentinel 会将主服务器实例结构 f1ags 属性的 SRI_S_DOWN标识打开，表示主服务器已经进人客观下线状态.

![image-20190930203258904](assets/Sentinel哨兵/image-20190930203258904.png)





**客观下线状态的判断条件**

当认为主服务器已经进入下线状态的 Sentinel 的数量，超过 Sentinel 配置中设置 的quorum 参数的值，那么诚 Sentinel 就会认为主服务器已经进入客观下线状态。比如说，如果 Sentinel 在启动时载入了以下配置

```
sentinel monitor master 127.0.0.1 2
```

那么包括当前 Sentinel 在内，只要总共有两个 Sentinel 认为主服务器已经进入下线状态，那么当前 Sentinel 就将主服务器判断为客观下线。

如果 Sentinel 在启动时载入了以下配置

```
sentinel monitor master 127.0.0.1 6379 5
```

那么包括当前 Sentinel 在内，总共要有五个 Sentinel 都认为主服务器已经下线，当前 Sentinel 才会将主服务器判断为客观下线。

**不同 Sentinel 判断客观下线的条件可能不同**

因为不同Sentinel 的配置可能不同.



## 选举领头Sentinel 

当一个主服务器被判断为客观下线时，监视这个下线主服务器的各个 Sentinel 会进行协

商，选举出一个领头 Sentinel，并由领头 Sentinel 对下线主服务器执行故障转移操作。

三个 Sentinel 都发现主服务器已经进人了客观下线状态,每个Sentinel 再次向其他 Sentinel 发送命令.

**先到先得**

如果接收到命令的 Sentinel 还没有设置局部领头 Sentinel 的话，它就会将发送方设为领头,并回复.

根据命令请求发送的先后顺序不同，可能会有某个 Sentinel 的 `SENTINEL is_master_down-by-addr `命令比起其他 Sentinel 发送的相同命令都更快到达，并最终胜出领头Sentinel 的选举，然后这个领头 Sentinel 就可以开始对主服务器执行故障转移操作了。



## 故障转移

在选举产生出领头 Sentinel 之后，领头 Sentinel 将对已下线的主服务器执行故障转移操作，该操作包含以下三个步骤：

1) 在已下线主服务器属下的所有从服务器里面，**挑选出一个从服务器，并将其转换为主服务器。**

2) **让已下线主服务器属下的所有从服务器改为复制新的主服务器。**

3) **将已下线主服务器设置为新的主服务器的从服务器，当这个旧的主服务器重新上线时，它就会成为新的主服务器的从服务器**



### 选出新的主服务器

故障转移操作第一步要做的就是在已下线主服务器属下的所有从服务器中，挑选出一个状态良好、数据完整的从服务器，然后向这个从服务器发送 `SLAVEOF no one `命令，将这个从服务器转换为主服务器。

**新的主服务器是怎样挑选出来的**

领头 Sentinel 会将已下绒主服务器的所有从服务器保存到一个列表里面，然后接照以下规则，一项一项地对列表进行过滤:

1) 删除列表中所有处于下线或者断线状态的从服务器，这可以保证列表中剩余的从服务器都是正常在线的。

2) 删除列表中所有最近五秒内没有回复过领头 Sentinel 的 INFO 命令的从服务器，这可以保证列表中剩余的从服务器都是最近成功进行过通信的。

3) 删除所有与已下线主服务器连接断开超过down-after-mill_seconds*10毫秒的从服务器：`down-after-millseconds `选项指定了判断主服务器下线所需的时间，而删除断开时长超过 `down- =after-millseconds`*10 毫秒的从服务器，则可以保证列表中剩余的从服务器都没有过早地与主服务器断开连接，换向话说，列表中剩余的从服务器保存的数据都是比较新的。

之后，领头 Sentinel 将根据从服务器的优先级，对列表中剩余的从服务器进行排序，并选出其中优先级最高的从服务器。

知果有多个具有相同最高优先级的从服务器，那么领头 Sentinel 将接照从服务器的复制偏移量，对具有相同最高优先级的所有从服务器进行排序，并选出其中偏移量最大的从服务器（复制偏移量最大的从履务器就是保存着最新数据的从服务器）。

最后，如果有多个优先级最高、复制偏移量最大的从服务器，那么领头 Sentinel 将接照运行ID 对这些从服务器进行排序，并选出其中运行 ID 最小的从服务器。



### 修改从服务器的复制目标

当新的主服务器出现之后，领头 Sentinel 下ー步要做的就是，**让已下线主服务器属下的所有从服务器去复制新的主服务器，这一动作可以通过向从服务器发送 SLAVEOF 命令来实现。**



![image-20190930204626235](assets/Sentinel哨兵/image-20190930204626235.png)

![image-20190930204636513](assets/Sentinel哨兵/image-20190930204636513.png)



### 将旧的主服务器变为从服务器

因为旧的主服务器已经下线，所以这种设置是保存在 server1 对应的实例结构里面的，当 server1 重新上线时，Sentinel 就会向它发送 SLAVEOF 命令，让它成为 server2 的从服务器。



## Raft

Sentinel 系统选举领头 Sentinel 的方法是对 Raft 算法的领头选举方法的实现，关于这方法的详细信息可以观看 Raf 算法的作者录制的”Raft教程”视频,或者 Raft算法的论文。

## 搭建

https://www.hi-linux.com/posts/36631.html

## 参考

https://www.cnblogs.com/kevingrace/p/9004460.html

