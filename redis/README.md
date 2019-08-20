### 缓存

#### 如何保证缓存与数据库的双写一致性
[参考文章](https://cloud.tencent.com/developer/article/1480733)

#### 什么是Redis
一个基于内存的高性能Key-Value 数据库  
#### 优点
速度快，基本上是内存数据库，定期将数据刷到硬盘中。  
#### 支持丰富的数据类型
String、 List、 Set、 Sorted Set、 Hash 。
底层的实现上，会根据存储的类型和数据长度自动的去选择实际存储的数据结构，比如String 就会有 数字和字符串类型，List 也会有两种存储数据结构。  
#### 持久化存储  
Redis 提供RDB 和 AOF 两种持久化方案。  RDB 是直接讲内存数据全量备份到文件中，而AOF则向文件后添加最新的更新指令等。
#### Redis 缺点
1. 因为是内存数据库所以单机和内存大小直接限制了它。所以要提前预估和节约内存。还有本身他支持集群模式，还有过期策略。
2. 如果进行全量同步需要生成RDB文件，并进行传输，会展用机器CPU还有消耗带宽。
3. 修改配置文件需要进行重启，需要将硬盘数据加载到内存，在这期间不能提供服务。  
#### 为什么Redis单线程模型效率高？
1. 村内村操作。
2. 核心是基于非阻塞的IO多路复用机制
3. 单线程避免了多线程的频繁上下文切换。
4. Redis使用Hash结构，读取速度快，对于一些其他的结构也会根据数据量采取不同的数据结构进行优化。

#### Redis是单线程的，如何提高多核CPU利用率？
可以在同一个服务器部署多个 Redis 的实例，并把他们当作不同的服务器来使用，在某些时候，无论如何一个服务器是不够的， 所以，如果你想使用多个 CPU ，你可以考虑一下分区。  

#### Redis 有几种持久化方式？
1. 全量 RDB持久化，是指再指定的时间间隔内将内存中的数据集快照写入磁盘，实际操作过程是，fork一个子进程，先将数据集写入临时文件，写入成功后再替换之前的文件，用二进制压缩储存
2. 增量 AOF持久化，以日志的形式记录服务器所处理的每一个写删除操作，查询操作不记录，以文本的方式记录，可以进行直接编辑查看。

**RDB 优缺点**
1. 优点
灵活设置备份频率和周期，你可能打算每小时归档一次最近24小时的数据，同时也要每天归档一次最近30甜的数据，通过这样的备份策略，可以做到灾备。  
非常适合冷备份，对于灾难恢复，RDB是非常不错的选择，因为我们可以非常轻松的讲一个单独的文件压缩后转移到其他的介质。  
性能最大化，在开始持久化时，它唯一需要做的只是 fork 出子进程，之后再由子进程完成这些持久化的工作，这样就可以极大的避免服务进程执行 IO 操作了。也就是说，RDB 对 Redis 对外提供的读写服务，影响非常小，可以让 Redis 保持高性能。  
恢复更快。相比于 AOF 机制，RDB 的恢复速度更更快，更适合恢复数据，特别是在数据集非常大的情况。  

2. 缺点
如果你想保证数据的高可用性，即最大限度的避免数据丢失，那么 RDB 将不是一个很好的选择。因为系统一旦在定时持久化之前出现宕机现象，此前没有来得及写入磁盘的数据都将丢失。  
由于 RDB 是通过 fork 子进程来协助完成数据持久化工作的，因此，如果当数据集较大时，可能会导致整个服务器停止服务几百毫秒，甚至是 1 秒钟。  

**AOF优缺点**
1. 优点
该机制可以带来更高的数据安全性，即数据持久性。Redis 中提供了 3 种同步策略，即每秒同步、每修改(执行一个命令)同步和不同步。  
  - 事实上，每秒同步也是异步完成的，其效率也是非常高的，所差的是一旦系统出现宕机现象，那么这一秒钟之内修改的数据将会丢失。
  - 而每修改同步，我们可以将其视为同步持久化，即每次发生的数据变化都会被立即记录到磁盘中。可以预见，这种方式在效率上是最低的。
  - 至于无同步，无需多言，我想大家都能正确的理解它。  

由于该机制对日志文件的写入操作采用的是 append 模式，因此在写入过程中即使出现宕机现象，也不会破坏日志文件中已经存在的内容。
  - 因为以 append-only 模式写入，所以没有任何磁盘寻址的开销，写入性能非常高
  - 另外，如果我们本次操作只是写入了一半数据就出现了系统崩溃问题，不用担心，在 Redis 下一次启动之前，我们可以通过 redis-check-aof 工具来帮助我们解决数据一致性的问题。

如果日志过大，Redis可以自动启用 rewrite 机制。即使出现后台重写操作，也不会影响客户端的读写。因为在 rewrite log 的时候，会对其中的指令进行压缩，创建出一份需要恢复数据的最小日志出来。再创建新日志文件的时候，老的日志文件还是照常写入。当新的 merge 后的日志文件 ready 的时候，再交换新老日志文件即可。

AOF 包含一个格式清晰、易于理解的日志文件用于记录所有的修改操作。事实上，我们也可以通过该文件完成数据的重建。

2. 缺点
对于相同数量的数据集而言，AOF 文件通常要大于 RDB 文件。RDB 在恢复大数据集时的速度比 AOF 的恢复速度要快。  
根据同步策略的不同，AOF 在运行效率上往往会慢于 RDB 。总之，每秒同步策略的效率是比较高的，同步禁用策略的效率和 RDB 一样高效。  
以前 AOF 发生过 bug ，就是通过 AOF 记录的日志，进行数据恢复的时候，没有恢复一模一样的数据出来。所以说，类似 AOF 这种较为复杂的基于命令日志/merge/回放的方式，比基于 RDB 每次持久化一份完整的数据快照文件的方式，更加脆弱一些，容易有 bug 。不过 AOF 就是为了避免 rewrite 过程导致的 bug ，因此每次 rewrite 并不是基于旧的指令日志进行 merge 的，而是基于当时内存中的数据进行指令的重新构建，这样健壮性会好很多。  


一般的持久化方案
主 AOF 从 RDB+AOF

#### redis 有几种过期策略
redis的过期策略就是指当Reddis中缓存的key过期了，Redis如何处理。  
Redis 提供了三种过期策略：
1. 被动删除：当读/写一个已经过期的key时，会触发惰性删除策略，直接删除掉这个过期的key  
2. 主动删除：由于惰性删除策略无法保证冷数据被即时删除，所以redis会定期主动淘汰一批已经过期的key
3. 主动删除：当前已用内存超过了设置的最大内存值，会触发主动清理策略。

#### Redis 有哪几种数据淘汰策略？
主要有六种淘汰策略：
1. volatile-lru
从已设置过期时间的数据集中挑选最近最少使用的数据淘汰。redis并不是保证取得所有数据集中最近最少使用的键值对，而只是随机挑选的几个键值对中的， 当内存达到限制的时候无法写入非过期时间的数据集。
2. volatile-ttl
从已设置过期时间的数据集中挑选将要过期的数据淘汰。redis 并不是保证取得所有数据集中最近将要过期的键值对，而只是随机挑选的几个键值对中的， 当内存达到限制的时候无法写入非过期时间的数据集。
3. volatile-random
从已设置过期时间的数据集中任意选择数据淘汰。当内存达到限制的时候无法写入非过期时间的数据集。
4. allkeys-lru
从数据集中挑选最近最少使用的数据淘汰。当内存达到限制的时候，对所有数据集挑选最近最少使用的数据淘汰，可写入新的数据集。
5. allkeys-random
从数据集中任意选择数据淘汰，当内存达到限制的时候，对所有数据集挑选随机淘汰，可写入新的数据集。
6. no-enviction
当内存达到限制的时候，不淘汰任何数据，不可写入任何数据集，所有引起申请内存的命令会报错。

如何选择淘汰策略？  
allkeys-lru：如果我们的应用对缓存的访问符合幂律分布，也就是存在相对热点数据，或者我们不太清楚我们应用的缓存访问分布状况，我们可以选择allkeys-lru策略。  
allkeys-random：如果我们的应用对于缓存key的访问概率相等，则可以使用这个策略。   
volatile-ttl：这种策略使得我们可以向Redis提示哪些key更适合被eviction。  
另外，volatile-lru策略和volatile-random策略适合我们将一个Redis实例既应用于缓存和又应用于持久化存储的时候，然而我们也可以通过使用两个Redis实例来达到相同的效果，值得一提的是将key设置过期时间实际上会消耗更多的内存，因此我们建议使用allkeys-lru策略从而更有效率的使用内存。

#### Redis回收进程如何工作。
- 一个客户端执行新的命令，添加新的数据
- Redis检查内存使用情况，如果大于设置的限制，则根据设定好的回收策略进行回收
- Redis执行命令

#### 如何防止大量的key同一时间一下子过期？
一般需要在事件上加一个随机值，使得过期事件分散一些。

#### Redis 的使用场景
1. 数据缓存
2. 会话缓存
3. 时效性数据
4. 访问频率
5. 计数器
6. 社交列表
7. 记录用户判定信息
8. 交集差集和并集
9. 热门列表于排行榜
10. 最新动态
11. 消息队列
12. 分布式锁

#### 请用 Redis 和任意语言实现一段恶意登录保护的代码，限制 1 小时内每用户 Id 最多只能登录 5 次。
使用列表，长度作为次数，里面存储事件，先去排除一下小于一个小时前的时间数据，然后做长度判断。

#### Redis实现分布式锁
1. setnx + 过期事件
2. redlock

#### redis 如何实现消息队列

#### 什么是Redis piplining?
一次请求/响应服务器能实现处理新的请求即使旧的请求还未被响应。这样就可以将多个命令发送到服务器，而不用等待回复，最后在一个步骤中读取该答复。

#### redis集群方案
可以使用广泛德尔 Redis Cluster  
Redis集群如何扩容
- 如果 Redis 被当做缓存使用，使用一致性哈希实现动态扩容缩容。
- 如果 Redis 被当做一个持久化存储使用，必须使用固定的 keys-to-nodes 映射关系，节点的数量一旦确定不能变化。否则的话(即Redis 节点需要动态变化的情况），必须使用可以在运行时进行数据再平衡的一套系统，而当前只有 Redis Cluster、Codis 可以做到这样。


#### 什么事Redis 主从同步
Redis 的主从同步机制，允许Slave 从Master  那里，通过网络传输拷贝到完整的数据备份，从而达到主从机制。  
- 主数据库可以进行读写操作，当发生写操作的时候自动将数据同步到从数据库，而从数据库一般是只读的，并接收主数据库同步过来的数据。
- 一个主数据库可以有多个从数据库，而一个从数据库只能有一个主数据库。
- 第一次同步时，主节点做一次 bgsave 操作，并同时将后续修改操作记录到内存 buffer ，待完成后将 RDB 文件全量同步到复制节点，复制节点接受完成后将 RDB 镜像加载到内存。加载完成后，再通知主节点将期间修改的操作记录同步到复制节点进行重放就完成了同步过程。

通过 Redis 的复制功，能可以很好的实现数据库的读写分离，提高服务器的负载能力。主数据库主要进行写操作，而从数据库负责读操作。

#### 如何使用Redis Cluster 实现高可用？
##### 说说Redis 哈希槽的概念？
Redis Cluster 没有使用一致性hash，而是引入了哈希槽对的概念。  
Redis机器哪又16384 个哈希槽，每个key通过CRC16校验后对 16384 取模来决定放置哪个槽，集群的每个节点负责一部分 hash 槽。  

##### Redis Cluster 的主从复制模型是怎样的？
为了使在部分节点失败或者大部分节点无法通信的情况下集群仍然可用，所以集群使用了主从复制模型，每个节点都会有 N-1 个复制节点。

所以，Redis Cluster 可以说是 Redis Sentinel 带分片的加强版。也可以说：

Redis Sentinel 着眼于高可用，在 master 宕机时会自动将 slave 提升为 master ，继续提供服务。
Redis Cluster 着眼于扩展性，在单个 Redis 内存不足时，使用Cluster 进行分片存储。

##### Redis Cluster 方案什么情况下会导致整个集群不可用？
有 A，B，C 三个节点的集群，在没有复制模型的情况下，如果节点 B 宕机了，那么整个集群就会以为缺少 5501-11000 这个范围的槽而不可用。  
[redis cluster 故障转移](https://enpsl.top/2019/01/25/2019-01-25-redis-cluster-out/)

#### Redis Cluster 会有写操作丢失吗？为什么？
Redis 并不能保证数据的强一致性，而是异步复制，这以为在实际中集群再特定的条件下会出现丢失写操作。  