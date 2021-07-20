#### 字符串

* Redis的字符串就是一个由字节组成的序列，它们和很多 编程语言里面的字符串没有什么明显的不同，跟C或者C卄风格的字符数组也相去不远。在Redis 里面，字符串可以存储以下3种类型的值。

  * 字节串(byte string )
  * 整数。
  * 浮点数。

* 用户可以通过给定一个任意的数值，对存储着整数或者浮点数的字符串执行自增(increment) 或者自减(decrement)操作，在有需要的时候，Redis还会将整数转换成浮点数。整数的取 值范围和系统的长整数(long integer )的取值范围相同(在32位系统上，整数就是32位 有符号整数，在64位系统上，整数就是64位有符号整数)，而浮点数的取值范围和精度则 与IEEE 754标准的双精度浮点数(double )相同。Redis明确地区分字节串、整数和浮点数 的做法是一种优势，比起只能够存储字节串的做法，Redis的做法在数据表现方面具有更大 的灵活性。

* Redis中的自增命令和自减命令

  | 命令        | 用例和描述                                                   |
  | ----------- | ------------------------------------------------------------ |
  | INCR        | INCR key-name	将键存储的值加上1                           |
  | DECR        | DECR key-name——将键存储的值减去1                             |
  | INCRBY      | INCRBY key-name amount	将键存储的值加上整数amount         |
  | DECRBY      | DECRBY key-name amount	将键存储的值减去整数amount         |
  | INCRBYFLOAT | INCRBYFLOAT key-name amount	将键存储的值加上浮点数amount,这个命令在Redis 2.6或以上的版本可用 |

* 当用户将一个值存储到Redis字符串里面的时候，如果这个值可以被解释(interpret)为十进 制整数或者浮点数，那么Redis会察觉到这一点，并允许用户对这个字符串执行各种INCR\*和 DECR\*操作。如果用户对一个不存在的键或者一个保存了空串的键执行自增或者自减操作，那么 Redis在执行操作时会将这个键的值当作是0来处理。如果用户尝试对一个值无法被解释为整数 或者浮点数的字符串键执行自增或者自减操作，那么Redis将向用户返回一个错误。

* 即使在设置键时输入的值为字符串，但只要这个值可以被解释为整数，我们就可以把它当作整数来处理。

* 除了自增操作和自减操作之外，Redis还拥有对字节串的其中一部分内容进行读取或者写入 的操作(这些操作也可以用于整数或者浮点数，但这种用法并不常见)

* 供Redis处理子串和二进制位的命令

  | 命令     | 用例和描述                                                   |
  | -------- | ------------------------------------------------------------ |
  | APPEND   | APPEND key-name value	将值value追加到给定键key-name当前存储的值的末尾 |
  | GETRANGE | GETRANGE key-name start end	获取一个由偏移量start至偏移量end范围内所有字符组成的子串，包括start和end在内 |
  | SETRANGE | SETRANGE key-name offset value	将从start偏移量开始的子串设置为给定值 |
  | GETBIT   | GETBIT key-name offset	将字节串看作是二进制位串(bit string),并返回位串中偏移量为offset的二进制位的值 |
  | SETBIT   | SETBIT key-name offset value——将字节串看作是二进制位串，并将位串中偏移量为 offset 的二进制位的值设置为value |
  | BITCOUNT | BITCOUNT key-name [start end]——统计二进制位串里面值为1的二进制位的数量，如果给定 了可选的start偏移量和end偏移量，那么只对偏移量指定范围内的二进制位进行统计 |
  | BITOP    | BITOP operation dest-key key-name [key-name ...]	对一个或多个二进制位串执行包括并(AND)、或(OR)、异或(XOR)、非(NOT)在内的任意一种按位运算操作(bitwise operation), 并将计算得出的结果保存在dest-key键里面 |

  *  Redis现在的GETRANGE命令是由以前的SUBSTR命令改名而来的

  * 在使用SETRANGE或者SETBIT命令对字符串进行写入的时候，如果字符串当前的长度不 能满足写入的要求，那么Redis会自动地使用空字节(null)来将字符串扩展至所需的长度，然 后才执行写入或者更新操作。
  * 在使用GETRANGE读取字符串的时候，超出字符串末尾的数据会被视为是空串，而在使用GETBIT读取二进制位串的时候，超出字符串末尾的二进制位会被视为是0。
  * 很多键值数据库只能将数据存储为普通的字符串，并且不提供任何字符串处理操作，有一些键 值数据库允许用户将字节追加到字符串的前面或者后面，但是却没办法像Redis 一样对字符串的子 串进行读写。从很多方面来讲，即使Redis只支持字符串结构，并且只支持本节列出的字符串处理 命令，Redis也比很多别的数据库要强大得多；通过使用子串操作和二进制位操作，配合WATCH命令、MULTI命令和EXEC命令，用户甚至可以自己动手去构建任何他们想要的数据结构。

#### 列表

* Redis的列表允许用户从序列的两端推入或者弹出元素，获取列表元 素，以及执行各种常见的列表操作。除此之外，列表还可以用来存储任务信息、最近浏览过的文 章或者常用联系人信息。

* —些常用的列表命令

  | 命令   | 用例和描述                                                   |
  | ------ | ------------------------------------------------------------ |
  | RPUSH  | RPUSH key-name value [value ...]	将一个或多个值推入列表的右端 |
  | LPUSH  | LPUSH key-name value [value ...]	将一个或多个值推入列表的左端 |
  | RPOP   | RPOP key-name—移除并返回列表最右端的元素                     |
  | LPOP   | LPOP key-name	移除并返回列表最左端的元素                  |
  | LINDEX | LINDEX key-name offset	返回列表中偏移量为offset的元素     |
  | LRANGE | LRANGE key-name start end	返回列表从start偏移量到end偏移量范围内的所有元素，其中偏移量为start和偏移量为end的元素也会包含在被返回的元素之内 |
  | LTRIM  | LTRIM key-name start end	对列表进行修剪，只保留从start偏移量到end偏移量范围内的元素，其中偏移量为start和偏移量为end的元素也会被保留 |

* 有几个列表命令可以将元素从一个列表移动到另一个列表，或者阻塞(block)执行命令的客户端直到有其他客户端给列表添加元素为止

  | 命令       | 用例和描述                                                   |
  | ---------- | ------------------------------------------------------------ |
  | BLPOP      | BLPOP key-name [keyname . . . ] timeout	从第一不非空列表中弹岀位于最左端的元素，或者在timeout秒之内阻塞并等待可弹出的元素出现 |
  | BRPOP      | BRPOP key-name [key-name . . . ] timeout	从第一个非空列表中弹出位于最右端的元素，或者在timeout秒之内阻塞并等待可弹出的元素出现 |
  | RPOPLPUSH  | RPOPLPUSH source-key dest-key  从source-key列表中弹出位于最右端的元素，然后 将这个元素推入dest-key列表的最左端，并向用户返回这个元素 |
  | BRPOPLPUSH | BRPOPLPUSH source-key dest-key timeout 从 source-key 列表中弹出位于最右端的元素，然后将这个元素推入dest-key 列表的最左端，并向用户返回这个元素；如果source-key 为空，那么在timeout秒之内阻塞并等待可弹出的元素出现 |

* 在Redis里面，多个命令原子地执行指的是，在这些命令正在读取或者修改数据的时候，其他客户端不能读取或者修改相同的数据。
* 对于阻塞弹出命令和弹出并推入命令，最常见的用例就是消息传递(messaging )和任务队 列(task queue)
* 通过列表来降低内存占用
  * 我们使用了有序集合来记录用户最近浏览过的商品，并把用户浏览这些商品时的时间戳设置为分值，从而使得程序可以在清理旧会话的过程中或是执行完购买操作之后，进行相应的数据分析。但由于保存时间戳需要占用相应的空间，所以如果分析操作并不需要用到时间戳的 话，那么就没有必要使用有序集合来保存用户最近浏览过的商品了。为此，请在保证语义不变的情况 下，将update_token()函数里面使用的有序集合替换成列表。
* 列表的一个主要优点在于它可以包含多个字符串值，这使得用户可以将数据集中在同一个地 方。Redis的集合也提供了与列表类似的特性，但集合只能保存各不相同的元素。

#### 集合

* Redis的集合以无序的方式来存储多个各不相同的元素，用户可以快速地对集合执行添加元素操作、移除元素操作以及检查一个元素是否存在于集合里。

* —些常用的集合命令

  | 命令        | 用例和描述                                                   |
  | ----------- | ------------------------------------------------------------ |
  | SADD        | SADD key-name item [item ...]	将一个或多个元素添加到集合里面，并返回被添加元素当中原本并不存在于集合里面的元素数量 |
  | SREM        | SREM key-name item [item ...]——从集合里面移除一个或多个元素，并返回被移除 元素的数量 |
  | SISMEMBER   | SISMEMBER key-name item	检查元素item是否存在于集合key-name里 |
  | SCARD       | SCARD key name	返回集合包含的兀素的数量                   |
  | SMEMBERS    | SMEMBERS key-name——返回集合包含的所有元素                    |
  | SRANDMEMBER | SRANDMEMBER key-name [count] 从集合里面随机地返回一个或多个元素。当count 为正数时，命令返回的随机元素不会重复；当count为负数时，命令返回的随机元素可能会 出现重复 |
  | SPOP        | SPOP key-name——随机地移除集合中的一个元素，并返回被移除的元素 |
  | SMOVE       | SMOVE source-key dest-key item 如果集合 source-key 包含元素 item,那么从 集合source-key里面移除元素item,并将元素item添加到集合dest-key中；如果item 被成功移除，那么命令返回1,否则返回0 |

* 用于组合和处理多个集合的Redis命令

  | 命令        | 用例和描述                                                   |      |
  | ----------- | ------------------------------------------------------------ | ---- |
  | SDIFF       | SDIFF key-name [key-name ...]	返回那些存在于第一个集合、但不存在于其他集合中的元素(数学上的差集运算) |      |
  | SDIFFSTORE  | SDIFFSTORE dest-key key-name [key-name ...]	将那些存在于第一个集合但并不存在于其他集合中的元素(数学上的差集运算)存储到dest-key键里面 |      |
  | SINTER      | SINTER key-name [key-name ...]	返回那些同时存在于所有集合中的元素(数学上的交集运算) |      |
  | SINTERSTORE | SINTERSTORE dest-key key-name [key-name ...]	将那些同时存在于所有集合的元素(数学上的交集运算)存储到dest-key键里面 |      |
  | SUNION      | SUNION key-name [key-name ...]	返回那些至少存在于一个集合中的元素(数学上的并集计算) |      |
  | SUNIONSTORE | SUNIONSTORE dest-key key-name [key-name ...]	将那些至少存在于一个集合中的元素(数学上的并集计算)存储到dest-key键里面 |      |

  * 这些命令分别是并集运算、交集运算和差集运算这3个基本集合操作的“返回结果”版本和 “存储结果”版本

#### 散列

* Redis的散列可以让用户将多个键值对存储到一个Redis键里面。从功能上 来说，Redis为散列值提供了一些与字符串值相同的特性，使得散列非常适用于将一些相关的数据存储在一起。我们可以把这种数据聚集看作是关系数据库中的行，或者文档数据库中的文档。

* 用于添加和删除键值对的散列操作

  | 命令  | 用例和描述                                                   |
  | ----- | ------------------------------------------------------------ |
  | HMGET | HMGET key-name key [key ...]	从散列里面获取一个或多个键的值 |
  | HMSET | HMSET key-name key value [key value ...]	为散列里面的一个或多个键设置值 |
  | HDEL  | HDEL key-name key [key ...]——删除散列里面的一个或多个键值对，返回成功找到并删除的 键值对数量 |
  | HLEN  | hlen key-name	返回散列包含的键值对数量                    |

* 展示Redis散列的更高级特性

  | 命令         | 用例和描述                                                   |
  | ------------ | ------------------------------------------------------------ |
  | HEXISTS      | HEXISTS key-name key——检査给定键是否存在于散列中             |
  | HKEYS        | HKEYS key-name	获取散列包含的所有键                       |
  | HVALS        | HVALS key-name	获取散列包含的所有值                       |
  | HGETALL      | HGETALL key-name——获取散列包含的所有键值对                   |
  | HINCRBY      | HINCRBY key-name key increment	将键 key 存储的值加上整数 increment |
  | HINCRBYFLOAT | HINCRBYFLOAT key-name key increment	将键 key 存储的值加上浮点数 increment |
  * 尽管有HGETALL存在，但HKEYS和HVALUES也是非常有用的:如果散列包含的值非常大, 那么用户可以先使用HKEYS取岀散列包含的所有键，然后再使用HGET 一个接一个地取出键的 值，从而避免因为一次获取多个大体积的值而导致服务器阻塞。
  * HINCRBY和HINCRBYFLOAT可能者回想起用于处理字符串的INCRBY和INCRBYFLOAT, 这两对命令拥有相同的语义，它们的不同在于HINCRBY和HINCRBYFLOAT处理的是散列，而不是字符串。
  * HINCRBY和HINCRBYFLOAT同字符串一样，对散列中一个尚未存在的'键执行自增操作时，Redis会将键的值当1作0来处理。

#### 有序集合

* 和散列存储着键与值之间的映射类似，有序集合也存储着成员与分值之间的映射，并且提供 了分值处理命令，以及根据分值大小有序地获取(fetch)或扫描(scan)成员和分值的命令。 

*  —些常用的有序集合命令

  | 命令    | 用例和描述                                                   |
  | ------- | ------------------------------------------------------------ |
  | ZADD    | ZADD key-name score member [score member ...]	将带有给定分值的成员添加到有序集合里面 |
  | ZREM    | ZREM key-name member [member ...]	从有序集合里面移除给定的成员，并返回被移除成员的数量 |
  | ZCARD   | ZCARD key-name	返回有序集合包含的成员数量                 |
  | ZINCRBY | ZINCRBY key-name increment member	将 member 成员的分值加上 increment |
  | ZCOUNT  | ZCOUNT key-name min max	返回分值介于min和max之间的成员数量 |
  | ZRANK   | ZRANK key-name member	返回成员member在有序集合中的排名    |
  | ZSCORE  | ZSCORE key-name member	返回成员 member 的分值             |
  | ZRANGE  | ZRANGE key-name start stop [WITHSCORES]	返回有序集合中排名介于 start 和 stop之间的成员，如果给定了可选的WITHSCORES选项，那么命令会将成员的分值也一并返回 |
  * 获取单个成员的分值对于实现 计数器或者排行榜之类的功能 非常有用。

* 有序集合的范围型数据获取命令和范围型数据删除命令，以及并集命令和交集命令

  | 命令             | 用例和描述                                                   |
  | ---------------- | ------------------------------------------------------------ |
  | ZREVRANK         | ZREVRANK key-name member	返回有序集合里成员member的排名，成员按照分值从大到小排列 |
  | ZREVRANGE        | ZREVRANGE key-name start stop [WITHSCORES]	返回有序集合给定排名范围内的成员，成员按照分值从大到小排列 |
  | ZRANGEBYSCORE    | ZRANGEBYSCORE key min max [WITHSCORES] [LIMIT offset count]	返回有序集合中，分值介于min和max之间的所有成员 |
  | ZREVRANGEBYSCORE | ZREVRANGEBYSCORE key max min [WITHSCORES] [LIMIT offset count]	获取有序集合中分值介于min和max之间的所有成员，并按照分值从大到小的顺序来返 回它们 |
  | ZREMRANGEBYRANK  | ZREMRANGEBYRANK key-name start stop	移除有序集合中排名介于 start 和 stop之间的所有成员 |
  | ZREMRANGEBYSCORE | ZREMRANGEBYSCORE key-name min max	移除有序集合中分值介于min和max之间的所有成员 |
  | ZINTERSTORE      | ZINTERSTORE dest-key key-count key [key ...] [WEIGHTS weight [weight ・..]] [AGGREGATE SUM \| MIN IMAX]——对给定的有序集合执行类似于集合的 交集运算 |
  | ZUNIONSTORE      | ZUNIONSTORE dest-key key-count key [key ...] [WEIGHTS weight \[weight ...]\[AGGREGATE SUM I MIN I MAX]——对给定的有序集合执行类似于集合的并集运算 |

  * ZREV\*命令的工作方式和相对应的非逆序命令的工作方式完全一样(逆序就是指 元素按照分值从大到小地排列)。
  * 用户可以在执行并 集运算和交集运算 的时候传人不同的 聚合函数，共有 sum、min、max 三个聚合函数可选。
    * 默认的聚合函数sum,所以输出有序集合成员的分值都 是通过加法计算得出的。
    * min 函数在多个输入有序集合都包含同一个成员的情况下，会将最小的那个分值设置为这个成员在输 出有序集合的分值。
    * max 函数在多个输入有序集合都包含同一个成员的情况下，会将最大的那个分值设置为这个成员在输 出有序集合的分值。
  * 用户还可以把集合作为输入传给ZINTERSTORE和ZUNIONSTORE, 命令会将集合看作是成员分值全为1的有序集合来处理。
  * 并集运算和交集运算不同，只要某个成员存在于至少一个输入有序集合里面，那么这个成员 就会被包含在输出有序集合里面。

#### 发布与订阅

* 一般来说，发布与订阅(又称pub/sub )的特 点是订阅者(listener)负责订阅频道(channel),发送者(publisher )负责向频道发送二进制字 符串消息(binary string message )0每当有消息被发送至给定频道时，频道的所有订阅者都会收 到消息。我们也可以把频道看作是电台，其中订阅者可以同时收听多个电台，而发送者则可以在 任何电台发送消息。

* Redis提供的发布与订阅命令

  | 命令         | 用例                                 | 描述                                                         |
  | ------------ | ------------------------------------ | ------------------------------------------------------------ |
  | PUBLISH      | PUBLISH channel message              | 向给定频道发送消息                                           |
  | PSUBSCRIBE   | PSUBSCRIBE pattern [pattern ..       | 订阅与给定模式相匹配的所有频道                               |
  | PUNSUBSCRIBE | PUNSUBSCRIBE [pattern [pattern ...]] | 退订给定的模式，如果执行时没有给定任何模式，那么退订所有模式 |
  | SUBSCRIBE    | SUBSCRIBE channel [channel ...]      | 订阅给定的一个或多个频道                                     |
  | UNSUBSCRIBE  | UNSUBSCRIBE [channel [channel ...]]  | 退订给定的一个或多个频道，如果执行时没 有给定任何频道，那么退订所有频道 |

* Redis系统的稳定性
  * 对于旧版Redis来说，如果一个客户端订阅了某个 或某些频道，但它读取消息的速度却不够快的话，那么不断积压的消息就会使得Redis输出缓冲 区的体积变得越来越大，这可能会导致Redis的速度变慢，甚至直接崩溃。也可能会导致Redis 被操作系统强制杀死，甚至导致操作系统本身不可用。新版的Redis不会出现这种问题，因为它 会自动断开不符合client-output-buffer-limit pub sub配置选项要求的订阅客户端
* 数据传输的可靠性
  * 任何网络系统在执行操作时都可能会遇上断线情况， 而断线产生的连接错误通常会使得网络连接两端中的其中一端进行重新连接。但是，如果客户端在执行订阅操作的过程中断线，那么 客户端将丢失在断线期间发送的所有消息，因此依靠频道来接收消息的用户可能会对Redis提供 的PUBLISH命令和SUBSCRIBE命令的语义感到失望。

#### 其他命令

* 将要介绍的命令则可以用于处理多种类型的数据:首先要介绍的是可以同时处理字符串、集合、列表和散列的SORT命令；之后要介绍是用于实现基本事务特性的MULT工命令和EXEC命令，这两个命令可以让用户将多个命令当作一个命令来执行；最后要介绍的是几个不同的自动过期命令，它 们可以自动删除无用数据。

##### 排序

* Redis的排序操作和其他编程语言的排序操作一样，都可以根据某种比较规则对一系列元素进行有序的排列。负责执行排序操作的SORT命令可以根据字符串、列表、集合、有序集合、散列这5种键里面存储着的数据，对列表、集合以及有序集合进行排序。如果读者之前曾经使用过 关系数据库的话，那么可以将SORT命令看作是SQL语言里的order by子句。

* SORT命令的定义

  | 命令 | 用例和描述                                                   |
  | ---- | ------------------------------------------------------------ |
  | SORT | SORT source-key [BY pattern] [LIMIT offset count] [GET pattern [GET pattern ..・]] [ASCI DESC] [ALPHA] [STORE dest-key] 根据给定的选项，对输入 列表、集合或者有序集合进行排序，然后返回或者存储排序的结果 |

* 使用SORT命令提供的选项可以实现以下功能:根据降序而不是默认的升序来排序元素； 将元素看作是数字来进行排序，或者将元素看作是二进制字符串来进行排序(比如排序字符 串，110,和的结果就跟排序数字110和12的结果不一样)；使用被排序元素之外的其 他值作为权重来进行排序，甚至还可以从输入的列表、集合、有序集合以外的其他地方进行 取值。

##### 基本的Redis事务

* 有时候为了同时处理多个结构，我们需要向Redis发送多个命令。尽管Redis有几个可以在两个键之间复制或者移动元素的命令，但却没有那种可以在两个不同类型之间移动元素的命令 (虽然可以使用ZUNIONSTORE命令将元素从一个集合复制到一个有序集合)。为了对相同或者不同类型的多个键执行操作，Redis有5个命令可以让用户在不被打断(interruption)的情况下对多个键执行操作，它们分别是WATCH、MULTI. EXEC、UNWATCH和DISCARD。
* 什么是Redis的基本事务
  * Redis的基本事务(basic transaction )需要用到MULTI命令和EXEC命令，这种事务可以让一个客户端在不被其他客户端打断的情况下执行多个命令。和关系数据库那种可以在执行的过程中进行回滚(rollback )的事务不同，在Redis里面，被MULTI命令和EXEC命令包围的所有命令会一个接一个地执行，直到所有命令都执行完毕为止。当一个事务执行完毕之后，Redis才会 处理其他客户端的命令。
  * 要在Redis里面执行事务，我们首先需要执行MULTI命令，然后输入那些我们想要在事务 里面执行的命令，最后再执行EXEC命令。当Redis从一个客户端那里接收到MULTI命令时， Redis会将这个客户端之后发送的所有命令都放入到一个队列里面，直到这个客户端发送EXEC 命令为止，然后Redis就会在不被打断的情况下，一个接一个地执行存储在队列里面的命令。
* 通过使用事务，各个线程都可 以在不被其他线程打断的情况下，执行各自队列里面的命令。记住，Redis要在接收到EXEC命 令之后，才会执行那些位于MULTI和EXEC之间的入队命令。
* 移除竞争条件
  * MULTI和EXEC事务的一个主要作用是移除竞争条件。
* 提高性能
  * 在Redis里面使用流水线的另一个目的是提高性能。 在执行一连串命令时，减少Redis与客户端之间的通信往返次数可以大幅降低客户端等待回复所需的 时间。

##### 键的过期时间

* 在使用Redis存储数据的时候，有些数据仅在一段很短的时间内有用，虽然我们可以在数据 的有效期过了之后手动删除无用的数据，但更好的办法是使用Redis提供的键过期操作来自动删 除无用数据。

* 在使用Redis存储数据的时候，有些数据可能在某个时间点之后就不再有用了，用户可以使 用DEL命令显式地删除这些无用数据，也可以通过Redis的过期时间(expiration )特性来让一个 键在给定的时限(timeout )之后自动被删除°当我们说一个键"带有生存时间(time to live )，r或 者一个键“会在特定时间之后过期(expire )”时，我们指的是Redis会在这个键的过期时间到达 时自动删除该键。

* 虽然过期时间特性对于清理缓存数据非常有用，不过对于列表、集合、散列和有序集合这样的容器(container)来说，键过期命令只能为整个键设 置过期时间，而没办法为键里面的单个元素设置过期时间(为了解决这个问题，本书在好几个地 方都使用了存储时间戳的有序集合来实现针对单个元素的过期操作)

* 用于处理过期时间的Redis命令

  | 命令      | 示例和描述                                                   |
  | --------- | ------------------------------------------------------------ |
  | PERSIST   | PERSIST key-name——移除键的过期时间                           |
  | TTL       | TTL key-name——查看给定键距离过期还有多少秒                   |
  | EXPIRE    | EXPIRE key-name seconds	让给定键在指定的秒数之后过期      |
  | EXPIREAT  | EXPIREAT key-name timestamp——将给定键的过期时间设置为给定的UNIX时间戳 |
  | PTTL      | PTTL key-name——査看给定键距离过期时间还有多少毫秒，这个命令在Redis 2.6或以上版本可用 |
  | PEXPIRE   | PEXPIRE key-name milliseconds	让给定键在指定的毫秒数之后过期，这个命令在Redis 2.6或以上版本可用 |
  | PEXPIREAT | PEXPIREAT key-name timestamp-milliseconds	将一个毫秒级精度的 UNIX 时间戳设置为给定键的过期时间，这个命令在Redis 2.6或以上版本可用 |