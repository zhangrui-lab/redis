Sentinel(哨岗、哨兵)是Redis的高可用性(high availability)解决方案:由一个或多个Sentinel实例(instance)组成的Sentinel系统(system)可以监视任意多个主服
务器, 以及这些主服务器属下的所有从服务器, 并在被监视的主服务器进入下线状态时，自动将下线主服务器属下的某个从服务器升级为新的主服务器，然后由新的主服务器代替已下线的主服务器继续处理命令请求。
* 令server1为监视下的一个主服务器。当server1的下线时长超过用户设定的下线时长上限时，Sentinel系统就会对server1执行故障转移操作:
  * 首先，Sentinel系统会挑选serverl属下的其中一个从服务器，并将这个被选中的从服务器升级为新的主服务器。
  * 之后，Sentinel系统会向server1属下的所有从服务器发送新的复制指令，让它们成为新的主服务器的从服务器，当所有从服务器都开始复制新的主服务器时，故障转移操作执行完毕。
  * 另外，Sentinel还会继续监视已下线的server1,并在它重新上线时，将它设置为新的主服务器的从服务器。

#### 启动并初始化Sentinel
* 启动一个Sentinel可以使用命令:`$ redis-sentinel /path/to/your/sentinel.conf`或者命令:`$ redis-server /path/to/your/sentinel.conf --sentinel`这两个命令的效果完全相同。
* 当一个Sentinel启动时，它需要执行以下步骤:
  1. 初始化服务器。
  2. 将普通Redis服务器使用的代码替换成Sentinel专用代码。
  3. 初始化Sentinel状态。
  4. 根据给定的配置文件，初始化Sentinel的监视主服务器列表。
  5. 创建连向主服务器的网络连接。

##### 初始化服务器
* 首先，因为Sentinel本质上只是一个运行在特殊模式下的Redis服务器，所以启动Sentinel的第一步，就是初始化一个普通的Redis服务器。不过，因为Sentinel执行的工作和普通Redis服务器执行的工作不同，所以Sentinel的初始化过程和普通Redis服务器的初始化过程并不完全相同。
* 例如，普通服务器在初始化时会通过载入RDB文件或者AOF文件来还原数据库状态，但是因为Sentinel并不使用数据库，所以初始化Sentinel时就不会载入RDB文件或者AOF文件。
* 下表展示了Redis服务器在Sentinel模式下运行时，服务器各个主要功能的使用情况

  | 功 能                                             | 使用情况                                                     |
  | ------------------------------------------------- | ------------------------------------------------------------ |
  | 数据库和键值对方面的命令，比如SET、DEL、FLUSHDB   | 不使用                                                       |
  | 事务命令，比如MULTI和WATCH                        | 不使用                                                       |
  | 脚本命令，比如EXEC。                               | 不使用                                                       |
  | RDB持久化命令，比如SAVE和BGSAVE                  | 不使用                                                       |
  | AOF持久化命令，比如BGREWRITEAOF                   | 不使用                                                       |
  | 复制命令，比如SLAVEOF                             | Sentinel内部可以使用，但客户端不可以使用                     |
  | 发布与订阅命令.比如PUBLISH和SUBSCRIBE             | SUBSCRIBE、PSUBSCRIBE、UNSUBSCRIBE、PUNSUBSCRIBE 四个命令在Sentinel内部和客户端都可以使用，但PUBLISH命令只能在Sentinel内部使用 |
  | 文件事件处理器(负责发送命令请求、处理命令回复) | Sentinel内部使用，但关联的文件事件处理器和普通Redis服务器不同 |
  | 时间事件处理器(负责执行serverCron函数)         | Sentinel内部使用，时间事件的处理器仍然是serverCron函数，serverCron函数会调用sentinel.c/sentinelTimer函数，后者包含了Sentinel要执行的所有操作 |

##### 使用Sentinel专用代码
* 启动Sentinel的第二个步骤就是将一部分普通Redis服务器使用的代码替换成Sentinel专用代码。比如说，普通Redis服务器使用redis.h/REDIS_SERVERPORT常量的值作为
服务器端口:`#define REDIS_SERVERPORT 6379` 而Sentinel则使用sentinel.c/REDIS_SENTINEL_PORT常量的值作为服务器端口 :`#define REDIS_SENTINEL_PORT 26379`
* 除此之外，普通Redis服务器使用redis.c/redisCommandTable作为服务器的命令表:
  ```
  struct redisCommand redisCommandTable[] = {
    {"get",getCommand,2,"r",0,NULL,1,1,1,0,01},
    { "set", setCommand, -3, "win", 0, noPreloadGetKeys, 1,1,1,0,0},
    { "setnx", setnxCommand, 3, "win", 0, noPreloadGetKeys, 1,1, lr 0,0 },
    // ...
    { "script", scriptcommand, -2, Hras'*, 0,NULL, 0, 0, 0,0, 0},
    {"bitop",bitopCommand,-4,"wm",0,NULL,2,-1,1,0,0},
    { "bitcount", bitcountCommand, -2, nrM, 0, NULL, 1,1,1,0,0}
  }
  ```
  而Sentinel则使用sentinel.c/sentinelcmds作为服务器的命令表，并且其中的 INFO 命令会使用Sentinel模式下的专用实现sentinel.c/sentinelInfoCorrunand函数，而不是普通Redis服务器使用的实现redis.c/infoCommand函数:
  ```
  struct redisCommand sentinelcmds[] = {
      { "ping', pingConimand, 1, n, 0, NULL, 0, 0, 0r 0, 0 },
      ('sentinel', sentinelCommand, -2,	0,NULL, 0, 0, 0, 0, 0},
      ("subscribe"，subscribeCommand,-2,0,NULL,0,0,0,0,0},
      {"unsubscribe",unsubscribeCommand,-1,nM,0,NULL,0,0,0,0, 0},
      ("psubscribe",psubscribeCommand,-2,nn,0,NULL,0,0,0,0,01,
      { "punsubscribe", punsubscribeCommand, -1, *'", 0, NULL, 0, 0, 0, 0,0 },
      { "INFO", InfoCommand, -1, 0,NULL, 0,0, 0, 0,0}
  }；
  ```
  sentinelcmds命令表也解释了为什么在Sentinel模式下，Redis服务器不能执行诸如 SET、DBSIZE等等这些命令，因为服务器根本没有在命令表中载入这些命令。PING、 SENTINEL、INFO、SUBSCRIBE、UNSUBSCRIBE、PSUBSCRIBE 和 PUNSUBSCRIBE 这七个命令就是客户端可以对Sentinel执行的全部命令了。

##### 初始化Sentinel状态
* 在应用了 Sentinel的专用代码之后，接下来，服务器会初始化一个sentinel.c/sentinelState结构(后面简称"Sentinel状态")，这个结构保存了服务器中所有和Sentinel功能有关的状态(服务器的一般状态仍然由redis.h/redisServer结构保存): 
  ```c
  struct sentinelstate (
    //当前纪元，用于实现故障转移
    uint64_t current_epoch;
    //保存了所有被这个sentinel监视的主服务器
    //字典的键是主服务器的名字
    //字典的值则是一个指向sentineRedisInstance结构的指针 
    diet *masters;
    //是否进入了 TILT 模式？
    int tilt;
    //目前正在执行的脚本的数量
    int running_scripts;
    //进入TILT模式的时间 
    mstime_t tilt_start_time;
    //最后一次执行时间处理器的时间 
    mstime_t previous_time;
    // 一个FIFO队列，包含了所有需要执行的用户脚本 
    list *scripts_queue;
  } sentinel;
  ```

##### 初始化Sentinel状态的masters属性
* Sentinel状态中的masters字典记录了所有被Sentinel监视的主服务器的相关信息，其中:
  * 字典的键是被监视主服务器的名字。
  * 而字典的值则是被监视主服务器对应的sentinel.c/sentinelRedisInstance结构。每个sentinelRedisInstance结构(后面简称“实例结构”)代表一个被Sentinel监视的Redis服务器实例(instance),这个实例可以是主服务器、从服务器，或者另外一个Sentinel。
* 实例结构包含的属性非常多，以下代码展示了实例结构在表示主服务器时使用的其中一部分属性
  ```c
  typedef struct sentinelRedisInstance (
    //标识值，记录了实例的类型，以及该实例的当前状态
    int flags;
    //实例的名字
    //主服务器的名字由用户在配置文件中设置
    //从服务器以及Sentinel的名字由Sentinel自动设置
    // 格式为 ip:port,例如 ”127.0.0.1:26379”
    char *name;
    //实例的运行
    char *runid;
    //配置纪元，用于实现故障转移
    uint64_t config_epoch;
    //实例的地址
    sentinelAddr *addr;
    // SENTINEL down-after-milliseconds 
    //实例无响应多少毫秒之后才会被判断为主观下线(subjectively down)
    mstime_t down_after_period;
    // SENTINEL monitor <master-name> <IP> <port> <quorum> 选项中的 quorum 参数 
    //判断这个实例为客观下线(objectively down )所需的支持投票数量 
    int quorum;
    // SENTINEL parallel-syncs <master-name>	选项的值
    //在执行故障转移操作时，可以同时对新的主服务器进行同步的从服务器数量
    int parallel_syncs;
    // SENTINEL failover-timeout <master-name>	选项的值
    //刷新故障迁移状态的最大时限
    mstime_t failover_timeout;
    // ...
  } sentinelRedisInstance;
  ```
* sentinelRedisInstance.addr 属性是—个指向 sentinel.c/sentinelAddr 结构的指针，这个结构保存着实例的IP地址和端口号:
  ```c
  typedef struct sentinelAddr (
    char *ip;
    int port;
  } sentinelAddr;
  ```
* 对Sentinel状态的初始化将引发对masters字典的初始化，而masters字典的初始化是根据被载入的Sentinel配置文件来进行的。

#### 获取主服务器信息
* Sentinel默认会以每十秒一次的频率，通过命令连接向被监视的主服务器发送命令，并通过分析**INFO**命令的回复来获取主服务器的当前信息。
* 通过分析主服务器返回的命令回复，Sentinel可以获取以下两方面的信息:
  * 一方面是关于主服务器本身的信息，包括run_id域记录的服务器运行ID,以及role域记录的服务器角色；
  * 另一方面是关于主服务器属下所有从服务器的信息，每个从服务器都由一个"slave" 字符串开头的行记录，每行的ip=域记录了从服务器的IP地址，而port=域则记录了从服务器的端口号。根据这些IP地址和端口号，Sentinel无须用户提供从服务器的地址信息，就可以自动发现从服务器。
* 根据run_id域和role域记录的信息，Sentinel将对主服务器的实例结构进行更新, 例如，主服务器重启之后，它的运行ID就会和实例结构之前保存的运行ID不同，Sentinel检测到这一情况之后，就会对实例结构的运行ID进行更新。
* 至于主服务器返回的从服务器信息，则会被用于更新主服务器实例结构的slaves字典，这个字典记录了主服务器属下从服务器的名单:
  * 字典的键是由Sentinel自动设置的从服务器名字，格式为ip:port:如对于IP地址为127.0.0.1,端口号为11111的从服务器来说，Sentinel为它设置的名字就是127.0.0.1:11111
  * 至于字典的值则是从服务器对应的实例结构:比如说，如果键是127.0.0.1:11111, 那么这个键的值就是IP地址为127.0.0.1,端口号为11111的从服务器的实例结构。
* Sentinel在分析INFO命令中包含的从服务器信息时，会检查从服务器对应的实例结构是否已经存在于slaves字典:
  * 如果从服务器对应的实例结构已经存在，那么Sentinel对从服务器的实例结构进行更新。
  * 如果从服务器对应的实例结构不存在，那么说明这个从服务器是新发现的从服务器， Sentinel会在slaves字典中为这个从服务器新创建一个实例结构。
* 注意对比图中主服务器实例结构和从服务器实例结构之间的区别:
  * 主服务器实例结构的flags属性的值为SRI_MASTER,而从服务器实例结构的flags属性的值为SRI_SLAVE。
  * 主服务器实例结构的name属性的值是用户使用Sentinel配置文件设置的，而从服务器实例结构的name属性的值则是Sentinel根据从服务器的1P地址和端口号自动设置的。

#### 获取从服务器信息
* 当Sentinel发现主服务器有新的从服务器出现时，Sentinel除了会为这个新的从服务器创建相应的实例结构之外，Sentinel还会创建连接到从服务器的命令连接和订阅连接。
* 在创建命令连接之后，Sentinel在默认情况下，会以每十秒一次的频率通过命令连接向从服务器发送INFO。
* 根据INFO命令的回复，Sentinel会提取出以下信息:
  * 从服务器的运行ID run_id
  * 从服务器的角色role
  * 主服务器的IP地址master_host,以及主服务器的端口号master_port
  * 主从服务器的连接状态master_link_status
  * 从服务器的优先级slave_priority
  * 从服务器的复制偏移量slave_repl_offset
  根据这些信息，Sentinel会对从服务器的实例结构进行更新

#### 向主服务器和从服务器发送信息
* 在默认情况下，Sentinel会以每两秒一次的频率，通过命令连接向所有被监视的主服务器和从服务器发送以下格式的命令:`PUBLISHsentinel:hello*'<s_ip>,<s_port>,<s_runid>,<s_epoch>,<m_name>,<m_ip>,<m_port>,<m_epoch>"`
* 这条命令向服务器的`__sentinel__: hello`频道发送了一条信息，信息的内容由多个参数组成:
  * 其中以`s_`开头的参数记录的是Sentinel本身的信息。
  * 而队`m_`开头的参数记录的则是主服务器的信息。如果Sentinel正在监视的是主服务器，那么这些参数记录的就是主服务器的信息；如果Sentinel正在监视的是从服务器，那么这些参数记录的就是从服务器正在复制的主服务器的信息。
* 信息中和Sentinel有关的参数

  | 参 数   | 意 义                                         |
  | ------- | --------------------------------------------- |
  | s_ip    | Sentinel的IP地址                           |
  | s_port  | Sentinel的端口号                              |
  | s_runid | Sentinel的运行ID                              |
  | s_epoch | Sentinel当前的配置纪元(configuration epoch) |
* 信息中和主服务器有关的参数

  | 参 数   | 意 义                  |
  | ------- | ---------------------- |
  | m_name  | 主服务器的名字         |
  | m_ip    | 主服务器的IP地址       |
  | m_port  | 主服务器的端口号       |
  | m_epoch | 主服务器当前的配置纪元 |
* 以下是一条Sentinel通过PUBLISH命令向主服务器发送的信息示例:`127.0.0.1,26379,e955b4c85598ef5b5f055bc7ebfd5e828dbed4fa,0,mymaster,127.0.0.1,6379,0` 这个示例包含了以下信息:
  *Sentinel的 IP地址为 127.0.0.1 端口号为 26379,运行 ID 为 e955b4c85598ef5b5f055bc7ebfd5e828dbed4fa,当前的配置纪元为 0。
  * 主服务器的名字为mymaster, IP地址为127.0.0.1,端口号为6379,当前的配置纪元为0

#### 接收来自主服务器和从服务器的频道信息
* 当Sentinel与一个主服务器或者从服务器建立起订阅连接之后，Sentinel就会通过订阅连接，向服务器发送以下命令:`SUBSCRIBE sentinel :hello` Sentinel对 
sentinel:hello 频道的订阅会一直持续到Sentinel与服务器的连接断开为止。这也就是说，对于每个与Sentinel连接的服务器，Sentinel既通过命令连接向服务器的 
sentinel:hello 频道发送信息，又通过订阅连接从服务器的 sentinel :hello 频道接收信息。
* 对于监视同一个服务器的多个Sentinel来说，一个 Sentinel发送的信息会被其他Sentinel接收到，这些信息会被用于更新其他Sentinel对发送信息Sentinel的认知，也会被用于更新其他Sentinel对被监视服务器的认知。
* 举个例子，假设现在有sentinel1, sentinel2, sentinel3 三个Sentinel在监视同一个服务器，那么当sentinel1向服务器的sentinel:hello频道发送一条信息时，所有订阅了sentinel:hello频道的Sentinel(包括sentinell自己在内)都会收到这条信息
* 当一个Sentinel从 \_sentinel_:hello频道收到一条信息时，Sentinel会对这条信息进行分析，提取出信息中的Sentinel IP地址、Sentinel 端口号、Sentinel 运行ID等八个参数，并进行以下检查:
  * 如果信息中记录的Sentinel运行ID和接收信息的Sentinel的运行ID相同，那么说明这条信息是Sentinel自己发送的，Sentinel将丢弃这条信息，不做进一步处理。
  * 相反地，如果信息中记录的Sentinel运行ID和接收信息的Sentinel的运行ID不相同，那么说明这条信息是监视同一个服务器的其他Sentinel发来的，接收信息的Sentinel将根据信息中的各个参数，对相应主服务器的实例结构进行更新。

##### 更新 sentinels 字典
* Sentinel为主服务器创建的实例结构中的sentinels字典保存了除Sentinel本身之外，所有同样监视这个主服务器的其他Sentinel的资料:
  * sentinels字典的键是其中一个Sentinel的名字，格式为ip:port。
  * sentinels字典的值则是键所对应Sentinel的实例结构。
* 当一个Sentinel接收到其他Sentinel发来的信息时(我们称呼发送信息的Sentinel为源Sentinel, 接收信息的Sentinel为目标Sentinel),目标Sentinel会从信息中分析并提取出以下两方面参数:
  * 与Sentinel有关的参数:源Sentinel的IP地址、端口号、运行ID和配置纪元。
  * 与主服务器有关的参数:源Sentinel正在监视的主服务器的名字、IP地址、端口号和配置纪元。
* 根据信息中提取出的主服务器参数，目标Sentinel会在自己的Sentinel状态的masters字典中査找相应的主服务器实例结构，然后根据提取出的Sentinel参数，检査主服务器实例结构的sentinels字典中，源Sentinel的实例结构是否存在:
  * 如果源Sentinel的实例结构已经存在，那么对源Sentinel的实例结构进行更新。
  * 如果源Sentinel的实例结构不存在，那么说明源Sentinel是刚刚开始监视主服务器的新Sentinel,目标Sentinel会为源Sentinel创建一个新的实例结构，并将这个结构添加到sentinels字典里面。
* 举个例子，假设分别有 `127.0.0.1:26379、127.0.0.1:26380、127.0.0.1:26381`  三个Sentinel正在监视主服务器 `127.0.0.1:6379` ,那么当 127.0.0.1:26379 这个Sentinel接收到以下信息时:
  ```
  "message"
  "__sentinel__:hello"
  "127.0.0.1,26379,e955b4c85598ef5b5f055bc7ebfd5e828dbed4fa,0,mymaster,127.0.0.1,6379,0"
  "message"
  "__sentinel__:hello"
  "127.0.0.1,26381,6241bf5cf9bfc8ecdl5d6eb6cc3185edfbb24903,0,mymaster,127.0.0.1,6379,0"
  "message**
  "__sentinel__:hello"
  "127.0.0.1,26380,a9b22fb79ae8fad28e4ea77d20398f77f6b89377,0,mymaster,127.0.0.1,6379,0”
  ```
  Sentinel将执行以下动作:
  * 第一条信息的发送者为127.0.0.1:26379自己，这条信息会被忽略。
  * 第二条信息的发送者为127.0.0.1:26381, Sentinel会根据这条信息中提取出的内容，对sentinels字典中127.0.0.1:26381对应的实例结构进行更新。
  * 第三条信息的发送者为127.0.0.1:26380, Sentinel会根据这条信息中提取出的内容，对sentinels字典中127.0.0.1:26380所对应的实例结构进行更新。
* 因为一个Sentinel可以通过分析接收到的频道信息来获知其他Sentinel的存在，并通过发送频道信息来让其他Sentinel知道自己的存在，所以用户在使用Sentinel的时候并不需要提供各个Sentinel的地址信息，监视同一个主服务器的多个Sentinel可以自动发现对方。

##### 创建连向其他Sentinel的命令连接
* 当Sentinel通过频道信息发现一个新的Sentinel时，它不仅会为新Sentinel在sentinels字典中创建相应的实例结构，还会创建一个连向新Sentinel的命令连接，
而新Sentinel也同样会创建连向这个Sentinel的命令连接，最终监视同一主服务器的多个Sentinel将形成相互连接的网络:SentinelA有连向SentinelB的命令连接，而SentinelB也有连向SentinelA的命令连接。
* 使用命令连接相连的各个Sentinel可以通过向其他Sentinel发送命令请求来进行信息交换。
* Sentinel之间不会创建订阅连接
  * Sentinel在连接主服务器或者从服务等时，会同时创建命令连接和订阅连接，但是在连接其他Sentinel时，只会创建命令连接，而不创建订阅连接。这是因为Sentinel需要通过接收
  主服务器或者从服务器发来的频道信息来发现未知的新Sentinel,所以才需要建立订阅连接，而相互已知的Sentinel只要使用命令连接来进行通信就足够了。

#### 检测主观下线状态
* 在默认情况下，Sentinel会以每秒一次的频率向所有与它创建了命令连接的实例(包括主服务器、从服务器、其他Sentinel在内)发送PING命令，并通过实例返回的PING命令回复来判断实例是否在线。
* 实例对PING命令的回复可以分为以下两种情况: 
  * 有效回复:实例返回 `+PONG、-LOADING, -MASTERDOWN` 三种回复的其中一种。
  * 无效回复:实例返回除+PONG、-LOADING、 -MASTERDOWN三种回复之外的其他回复，或者在指定时限内没有返回任何回复。
*Sentinel配置文件中的`down-after-milliseconds`选项指定了Sentinel判断实例进入主观下线所需的时间长度:如果一个实例在down-after-milliseconds毫秒内，
连续向Sentinel返回无效回复，那么Sentinel会修改这个实例所对应的实例结构，在结构的flags属性中打开SRI_S_DOWN标识，以 此来表示这个实例已经进入主观下线状态
  * 如果配置文件指定Sentinel1的down-after-milliseconds选项的值为50000毫秒，那么当主服务器master连续50000毫秒都向Sentinel1返回无效回复时，Sentinel1就会将master标记为主观下线，并在master所对应的实例结构的flags属性中打开SRI_S_DOWN标识。
* 主观下线时长选项的作用范围
  * 用户设置的down-after-milliseconds选项的值，不仅会被Sentinel用来判断主服务器的主观下线状态，还会被用于判断主服务器属下的所有从服务器。举个例子，如果用户向Sentinel设置了以下配置:
    ```
    sentinel monitor master 127.0.0.1 6379 2
    sentinel down-after-milliseconds master 50000
    ```
    那么50000毫秒不仅会成为Sentinel判断master进入主观下线的标准，还会成为 Sentinel判断master属下所有从服务器，以及所有同样监视master的其他Sentinel进入主观下线的标准。
* 多个Sentinel设置的主观下线时长可能不同
  * down-after-milliseconds选项另一个需要注意的地方是，对于监视同一个主服务器的多个Sentinel来说，这些Sentinel所设置的down-after-milliseconds
  选项的值也可能不同，因此，当一个Sentinel将主服务器判断为主观下线时，其他Sentinel可能仍然会认为主服务器处于在线状态。举个例子，如果Sentinel1载入了以下配置:
    ```
    sentinel monitor master 127.0.0.1 6379 2
    sentinel down-after-milliseconds master 50000
    ```
    而 Sentinel2 则载入了以下配置:
    ```
    sentinel monitor master 127.0.0.1 6379 2
    sentinel down-after-milliseconds master 10000
    ```
    那么当master的断线时长超过10000毫秒之后，Sentinel2会将master判断为主观下线，而Sentinel1却认为master仍然在线。只有当master的断线时长超过50000毫秒之后，Sentinel1和Sentinel2才会都认为master进入了主观下线状态。

#### 检查客观下线状态
* 当Sentinel将一个主服务器判断为主观下线之后，为了确认这个主服务器是否真的下线了，它会向同样监视这一主服务器的其他Sentinel进行询问，看它们是否也
认为主服务器已经进入了下线状态(可以是主观下线或者客观下线)。当Sentinel从其他Sentinel那里接收到足够数量的已下线判断之后，Sentinel就会将从服务器判定为客观下线，并对主服务器执行故障转移操作。

##### 发送 SENTINEL is-master-down-by-addr 命令
*Sentinel使用:`SENTINEL is-master-down-by-addr <ip> <port> <current_epoch> <runid>` 命令询问其他Sentinel是否同意主服务器已下线，命令中的各个参数的意义如表所示

  | 参 数         | 意 义                                                        |
  | ------------- | ------------------------------------------------------------ |
  | ip            | 被Sentinel判断为主观下线的主服务器的IP地址                   |
  | port          | 被Sentinel判断为主观下线的主服务器的端口号                   |
  | current_epoch | Sentinel当前的配置纪元，用于选举领头Sentinel                 |
  | runid         | 可以是*符号或者Sentinel的运行ID: *符号代表命令仅仅用于检测主服务器的客观下线状态，而Sentinel的运行ID则用于选举领头Sentinel |
* 举个例子，如果被Sentinel判断为主观下线的主服务器的IP为127.0.0.1,端口号为6379,并且Sentinel当前的配置纪元为0,那么Sentinel将向其他Sentinel发送以下命令:`SENTINEL is-master-down-by-addr 127.0.0.1 6379 0 *`

##### 接收 SENTINEL is-master-down-by-addr 命令
* 当一个Sentinel(目标Sentinel)接收到另一个Sentinel(源Sentinel)发来的`SENTINEL is-master-down-by`命令时，目标Sentinel会分析并取出命令
请求中包含的各个参数，并根据其中的主服务器IP和端口号，检査主服务器是否已下线，然后向源Sentinel返回一条包含三个参数的`Multi Bulk`回复作为SENTINEL is-master-down-by命令的回复:
  * <down_state>
  * <leader_runid>
  * <leader_epoch>
* SENTINEL is-master-down-by-addr 回复的意义

  | 参 数        | 意 义                                                        |
  | ------------ | ------------------------------------------------------------ |
  | down_state   | 返回目标Sentinel对主服务器的检査结果，1代表主服务器已下线，0代表主服务器未下线 |
  | leader_runid | 可以是*符号或者目标Sentinel的局部领头Sentinel的运行ID: *符号代表命令仅仅用于检测主服务器的下线状态，而局部领头Sentinel的运行ID则用于选举领头Sentinel |
  | leader_epoch | 目标Sentinel的局部领头Sentinel的配置纪元，用于选举领头Sentinel。仅在leader_runid的值不为\*时有效，如果leader_runid的值为\*,那么leader_epoch总为0 |
* 举个例子，如果一个Sentinel返回以下回复作为SENTINEL is-master-down-by-addr命令的回复:
  ```
  1) 1
  2) *
  3) 0
  ```
  那么说明Sentinel也同意主服务器已下线。

##### 接收 SENTINEL is-master-down-by-addr 命令的回复
* 根据其他Sentinel发回的`SENTINEL is-master-down-by-addr`命令回复，Sentinel将统计其他Sentinel同意主服务器已下线的数量，当这一数量达到配
置指定的判断客观下线所需的数量时，Sentinel会将主服务器实例结构flags属性的SRI_O_DOWN标识打开，表示主服务器已经进入客观下线状态。
* 客观下线状态的判断条件
  * 当认为主服务器已经进入下线状态的Sentinel的数量，超过Sentinel配置中设置的`quorum`参数的值，那么该Sentinel就会认为主服务器已经进入客观下线
  状态。比如说， 如果Sentinel在启动时载入了以下配置:`sentinel monitor master 127.0.0.1 6379 2` 那么包括当前Sentinel在内，只要总共有两个
  Sentinel认为主服务器已经进入下线状态，那么当前Sentinel就将主服务器判断为客观下线。又比如说，如果Sentinel在启动时载入了以下配置:`sentinel monitor master 127.0.0.1 6379 5` 
  那么包括当前Sentinel在内，总共要有五个Sentinel都认为主服务器已经下线，当前Sentinel才会将主服务器判断为客观下线。
* 不同Sentinel判断客观下线的条件可能不同
  * 对于监视同一个主服务器的多个Sentinel来说，它们将主服务器标判断为客观下线的条件可能也不同: 当一个Sentinel将主服务器判断为客观下线时，其他Sentinel可能
  并不是那么认为的。比如说，对于监视同一个主服务器的五个Sentinel来说，如果Sentinel1在启动时载入了以下配置: `sentinel monitor master 127.0.0.1 6379 2` 
  那么当五个Sentinel中有两个Sentinel认为主服务器已经下线时，Sentinel1就会将主服务器标判断为客观下线。而对于载入了以下配置的Sentinel2来说:
  `sentinel monitor master 127.0.0.1 6379 5` 仅有两个Sentinel认为主服务器已下线，并不会令 Sentinel2 将主服务器判断为客观下线。

##### 选举领头Sentinel
* 当一个主服务器被判断为客观下线时，监视这个下线主服务器的各个Sentinel会进行协商，选举出一个领头Sentinel, 并由领头Sentinel对下线主服务器执行故障转移操作。 以下是Redis选举领头Sentinel的规则和方法:
  * 所有在线的Sentinel都有被选为领头Sentinel的资格，换句话说，监视同一个主服务器的多个在线Sentinel中的任意一个都有可能成为领头Sentinel
  * 每次进行领头Sentinel选举之后，不论选举是否成功，所有Sentinel的配置纪元(configuration epoch)的值都会自增一次。配置纪元实际上就是一个计数器，并没有什么特别的。
  * 在一个配置纪元里面，所有Sentinel都有一次将某个Sentinel设置为局部领头Sentinel的机会，并且局部领头一旦设置，在这个配置纪元里面就不能再更改。
  * 每个发现主服务器进入客观下线的Sentinel都会要求其他Sentinel将自己设置为局部领头Sentinel。
  * 当一个Sentinel(源Sentinel)向另一个Sentinel(目标Sentinel)发送SENTINEL is-master-down-by-addr命令，并且命令中的runid参数不是\*符号而是源Sentinel的运行ID时，这表示源Sentinel要求目标Sentinel将前者设置为后者的局部领头Sentinel 
  * Sentinel设置局部领头Sentinel的规则是先到先得:最先向目标Sentinel发送设置要求的源Sentinel将成为目标Sentinel的局部领头Sentinel,而之后接收到的所有设置要求都会被目标Sentinel拒绝。
  * 目标Sentinel在接收到SENTINEL is-master-down-by-addr命令之后，将向源Sentinel返回一条命令回复，回复中的leader_runid参数和leader_epoch参数分别记录了目标Sentinel的局部领头Sentinel的运行ID和配置纪元。
  * 源Sentinel在接收到目标Sentinel返回的命令回复之后，会检查回复中leader_epoch参数的值和自己的配置纪元是否相同，如果相同的话，那么源Sentinel继续取出回复中的leader_runid参数，如果leader_runid参数的值和源Sentinel的运行ID一致，那么表示目标Sentinel将源Sentinel设置成了局部领头Sentinel。
  * 如果有某个Sentinel被半数以上的Sentinel设置成了局部领头Sentinel,那么这个Sentinel成为领头Sentinel
    * 举个例子，在一个由10个Sentinel组成的Sentinel系统里面，只要有大于等于10/2+1=6个Sentinel将某个Sentinel设置为局部领头Sentinel,那么被设置的那个Sentinel就会成为领头Sentinel。
  * 因为领头Sentinel的产生需要半数以上Sentinel的支持，并且每个Sentinel在每个配置纪元里面只能设置一次局部领头Sentinel,所以在一个配置纪元里面，只会出现一个领头Sentinel
  * 如果在给定时限内，没有一个Sentinel被选举为领头Sentinel,那么各个Sentinel将在一段时间之后再次进行选举，直到选出领头Sentinel为止。
* 为了熟悉以上规则，让我们来看一个选举领头Sentinel的过程。
  * 假设现在有三个Sentinel正在监视同一个主服务器，并且这三个Sentinel之前已经通过`SENTINEL is-master-down-by-addr`命令确认主服务器进入了
  客观下线状态。那么为了选出领头Sentinel, 三个Sentinel将再次向其他Sentinel发送`SENTINEL is- master-down-by-addr`命令。和检测客观下线状
  态时发送的`SENTINEL is-master-down-by-addr`命令不同， Sentinel这次发送的命令会带有Sentinel自己的运行ID,例如:`SENTINEL is-master-down-by-addr 127.0.0.1 6379 0 e955b4c85598ef5b5f055bc7ebfd5e828dbed4fa`
  如果接收到这个命令的Sentinel还没有设置局部领头Sentinel的话，它就会将运行 ID 为 e955b4c85598ef5b5f055bc7ebfd5e828dbed4fa 的Sentinel设置为自己的局部领头Sentinel,
  并返回类似以下的命令回复:`1 e955b4c85598ef5b5f055bc7ebfd5e828dbed4fa 0`然后接收到命令回复的Sentinel就可以根据这一回复，统计出有多少个Sentinel将自己设置成了局部领头Sentinel
  * 根据命令请求发送的先后顺序不同，可能会有某个Sentinel的SENTINEL is-master-down-by-addr命令比起其他Sentinel发送的相同命令都更快到达，并最终胜出领头Sentinel的选举，然后这个领头Sentinel就可以开始对主服务器执行故障转移操作了。

#### 故障转移
* 在选举产生出领头Sentinel之后，领头Sentinel将对已下线的主服务器执行故障转移操作，该操作包含以下三个步骤:
  1. 在已下线主服务器属下的所有从服务器里面，挑选出一个从服务器，并将其转换为主服务器。
  2. 让已下线主服务器属下的所有从服务器改为复制新的主服务器。
  3. 将已下线主服务器设置为新的主服务器的从服务器，当这个旧的主服务器重新上线时，它就会成为新的主服务器的从服务器。

##### 选出新的主服务器
* 故障转移操作第一步要做的就是在已下线主服务器属下的所有从服务器中，挑选出一个状态良好、数据完整的从服务器，然后向这个从服务器发送`SLAVEOF no one`命令,将这个从服务器转换为主服务器。
* 新的主服务器是怎样挑选出来的
  * 领头Sentinel会将已下线主服务器的所有从服务器保存到一个列表里面，然后按照以下规则，一项一项地对列表进行过滤:
    1. 删除列表中所有处于下线或者断线状态的从服务器，这可以保证列表中剩余的从服务器都是正常在线的。
    2. 删除列表中所有最近五秒内没有回复过领头Sentinel的INFO命令的从服务器，这可以保证列表中剩余的从服务器都是最近成功进行过通信的。
    3. 删除所有与已下线主服务器连接断开超过`down-after-milliseconds * 10`毫秒的从服务器:down-after-milliseconds选项指定了判断主服务器下
    线所需的时间，而删除断开时长超过down-after-milliseconds * 10毫秒的从服务器，则可以保证列表中剩余的从服务器都没有过早地与主服务器断开连接，换句话说，列表中剩余的从服务器保存的数据都是比较新的。
  * 领头Sentinel将根据从服务器的优先级，对列表中剩余的从服务器进行排序，并选出其中优先级最高的从服务器。
  * 如果有多个具有相同最高优先级的从服务器，那么领头Sentinel将按照从服务器的复制偏移量，对具有相同最高优先级的所有从服务器进行排序，并选出其中偏移量最大的从服务器(复制偏移量最大的从服务器就是保存着最新数据的从服务器)。
  * 最后，如果有多个优先级最高、复制偏移量最大的从服务器，那么领头Sentinel将按照运行ID对这些从服务器进行排序，并选出其中运行ID最小的从服务器。
* 在发送`SLAVEOF no one`命令之后，领头Sentinel会以每秒一次的频率(平时是每十秒一次)，向被升级的从服务器发送INFO命令，并观察命令回复中的角色(role)信息，当被升级服务器的role从原来的slave变为master时，领头 Sentinel就知道被选中的从服务器已经顺利升级为主服务器了。

##### 修改从服务器的复制目标
* 当新的主服务器出现之后，领头Sentinel下一步要做的就是，让已下线主服务器属下的所有从服务器去复制新的主服务器，这一动作可以通过向从服务器发送SLAVEOF命令来实现。

##### 将旧的主服务器变为从服务器
* 故障转移操作最后要做的是，将已下线的主服务器设置为新的主服务器的从服务器。 

#### 重点回顾
* Sentinel只是一个运行在特殊模式下的Redis服务器，它使用了和普通模式不同的命令表，所以Sentinel模式能够使用的命令和普通Redis服务器能够使用的命令不同。
* Sentinel会读入用户指定的配置文件，为每个要被监视的主服务器创建相应的实例结构，并创建连向主服务器的命令连接和订阅连接，其中命令连接用于向主服务器发送命令请求，而订阅连接则用于接收指定频道的消息。
* Sentinel通过向主服务器发送命令来获得主服务器属下所有从服务器的地址信息，并为这些从服务器创建相应的实例结构，以及连向这些从服务器的命令连接和订阅连接。
* 在一般情况下，Sentinel以每十秒一次的频率向被监视的主服务器和从服务器发送 INFO命令,当主服务器处于下线状态，或者Sentinel正在对主服务器进行故障转移操作时，Sentinel向从服务器发送INFO命令的频率会改为每秒一次。
* 对于监视同一个主服务器和从服务器的多个Sentinel来说，它们会以每两秒一次的频率，通过向被监视服务器的sentinel:hello频道发送消息来向其他 Sentinel宣告自己的存在。
* 每个Sentinel也会从sentinel:hello频道中接收其他Sentinel发来的信息，并根据这些信息为其他Sentinel创建相应的实例结构，以及命令连接。
* Sentinel只会与主服务器和从服务器创建命令连接和订阅连接，Sentinel与Sentinel之间则只创建命令连接。
* Sentinel以每秒一次的频率向实例(包括主服务器、从服务器、其他Sentinel)发送PING命令,并根据实例对PING命令的回复来判断实例是否在线，当一个实例在指定的时长中连续向Sentinel发送无效回复时，Sentinel会将这个实例判断为主观下线。
* 当Sentinel将一个主服务器判断为主观下线时，它会向同样监视这个主服务器的其他Sentinel进行询问，看它们是否同意这个主服务器已经进入主观下线状态。
* 当Sentinel收集到足够多的主观下线投票之后，它会将主服务器判断为客观下线，并发起一次针对主服务器的故障转移操作。
