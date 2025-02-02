# 常见数据问题

###1. 布隆过滤器是什么？

对一个值，进行多次hash，然后将对应位置的hash值置为1，就代表着个值，同时查询的时候计算多次，然后只要有一个位置为0，说明就是不存在。

### 2. redis 常用的数据结构【2月18好未来】【2月20滴滴】

- string 可以用来计数，缓存，分布式锁
- hash 可以用来保存用户的一些属性信息，用户的详情页
- list 可以用来做队列,可以用来做栈，可以用来做数据， 可以维护一个评论列表。lrange区间操作。
- set 可以用来做 交集 并集 差集，微博抽奖，随机事件问题。无序、去重
- sorted set  可以用来做排行榜，带权重的队列。
- bitmap  用来保存用户的登录信息，可以查询最近几个月的登录情况：bitop 可以用来做and or意味着有更多的选择。
- pub/sub 发布订阅
- stream
- hyperloglog  布隆过滤器器（滑动窗口）

### 3. redis 底层存储也是使用的字典

也就是hashmap，基本随机性比较大，然后很少有冲突。

### 4. 跳跃表的时间复杂度

一般是O(logn)  最差是 O(n)

# Redis 高性能、高并发

### 1. redis 为什么快

- 因为全部是内存操作，纯是内存操作，查找和操作的时间都是O(1)
- 单线程操作避免了上下文切换。不存在多线程和多进程导致的切换和消耗CPU。不用考虑各种锁问题
- 可以将一些事务由并行改为串行。
- 使用的是非阻塞io，
- 是使用自己的特定的数据结构，且数据结构比较简单，
- 完全基于内存，绝大部分请求是纯粹的内存操作，非常快速。
- 它的数据存在内存中，类似于HashMap，HashMap的优势就是查找和操作的时间复杂度都是O(1)；

### 2. redis的扩容和扩容是怎样实现的，是否会阻塞

因为插入和删除，当数量太多或者太少的时候，就会进行rehash，为了让hash表达的负载因子在一个合理的范围内。需要扩展时，需要将ht[1] = ht[0].used *2 的 2^n。需要收缩时，需要将ht[1]= ht[0].used*2 的2^n。然后将 ht[0]  置为空，ht1[1] = ht[0]， 然后再创建新的ht[1] 为下一次rehash做准备。

不是一次性，集中式的执行，而是分批次，渐进执行
因为如果键值数量过大的话，会导致阻塞，在一段时间内不能提供服务
采用的是分而治之的方式来执行的
索引期间，查询的话，会先在ht[0] 上面查询，然后新家的话，只会在ht[1]上面新加，这样的话，会保证ht[0] 上面的数据只增不减

### 3. 哈希表的扩展与收缩

当没有bgsave 和BGREWRITEAOF 的时候，负载因子为1。当有bgsave 和bswriteaof的时候，负载因为大于5。负载因子 = 哈希表已保存节点数量 / 哈希表大小load_factor = ht[0].used / ht[0].size。

### 4. redis过期提供了几种可选策略

- noeviction 不会继续服务写请求 (DEL 请求可以继续服务)，读请求可以继续进行。这样可以保证不会丢失数据，但是会让线上的业务不能持续进行。这是默认的淘汰策略。
- volatile-lru 尝试淘汰设置了过期时间的 key，最少使用的 key 优先被淘汰。没有设置过期时间的 key 不会被淘汰，这样可以保证需要持久化的数据不会突然丢失。
- volatile-ttl 跟上面一样，除了淘汰的策略不是 LRU，而是 key 的剩余寿命 ttl 的值，ttl 越小越优先被淘汰。
- volatile-random 跟上面一样，不过淘汰的 key 是过期 key 集合中随机的 key。
- allkeys-lru 区别于 volatile-lru，这个策略要淘汰的 key 对象是全体的 key 集合，而不只是过期的 key 集合。这意味着没有设置过期时间的 key 也会被淘汰。
- allkeys-random 跟上面一样，不过淘汰的策略是随机的 key。

### 5. redis的LRU删除机制

LRU 淘汰不一样，它的处理方式只有懒惰处理。当 Redis 执行写操作时，发现内存超出 maxmemory，就会执行一次 LRU 淘汰算法。

这个算法也很简单，就是随机采样出 5(可以配置) 个 key，然后淘汰掉最旧的 key，如果淘汰后内存还是超出 maxmemory，那就继续随机采样淘汰，直到内存低于 maxmemory 为止。

### 6. ROB的原理

copy on write。父进程会fork一个子进程，父进程和子进程共享内存空间。父进程继续提供读写服务，脏数据会继续和子进程区分开来。

你给出两个词汇就可以了，fork和cow。fork是指redis通过创建子进程来进行RDB操作，cow指的是copy on write，子进程创建后，父子进程共享数据段，父进程继续提供读写服务，脏数据会逐渐和子进程分离开来。

### 7. redis rdb持久化方式

- save 900 1  900秒（15分钟）内有1个更改就同步
- save 300 10 300秒（5分钟）内有10个更改就同步
- save 60 10000   60秒内有10000个更改就同步

### 8. redis aof 持久化机制

- appendfsync=always 每次同步 安全最高，最多丢失一个写入的数据
- appendfsync=everysec 每秒同步 安全最折中，最多丢失1S的数据
- appendfsync=no 安全最低，最多上一次保存AOF文件到当前时刻的全部数据

### 9. redis 的持久化, rdb 和aof的区别

rdb 是做的全量的快照，rdb是fork一个子进程去做持久化的。可以做冷备份，一旦挂了想恢复多久之前的数据，直接拷贝一份就好了。如果丢失数据的话，丢失数据还是比较多的。

aof 就是做增量添加。可以设置持久化的频率。也可以手动在低频率请求的时候做持久化。后台一秒执行一次fsync的话，丢失的最多也就一秒的的数据。aof是一个非常可读的数据。可以通过修改aof文件，来恢复之前的数据。

redis在重启的时候，会优先使用aof，因为aof要比rdb完整。rdb丢数据的话，丢失的数据还是比较多的。aof的话，丢数据的话，丢失的还是比较少的。

### 10. 主从如何同步

当启动一个slave的时候，他发送psync给master。如果是首次连接master，那么master会启动一个线程，进行全量的rdb快照，然后发送给slave。然后把新的请求写到缓存里面，然后slave会执行rdb文件，然后写到自己的本地。然后再读取master里面新增的请求。


### 11. redis 的集群。

redis cluster 着眼于可扩展。当单个redis不足时，使用cluster进行分片存储。。

Redis 支持主从同步，提供 Cluster 集群部署模式，通过 Sentine l哨兵来监控 Redis 主服务器的状态。当主挂掉时，在从节点中根据一定策略选出新主，并调整其他从 slaveof 到新主。

选主的策略简单来说有三个：
slave 的 priority 设置的越低，优先级越高；
同等情况下，slave 复制的数据越多优先级越高；
相同的条件下 runid 越小越容易被选中。
在 Redis 集群中，sentinel 也会进行多实例部署，sentinel 之间通过 Raft 协议来保证自身的高可用。

### 12. redis的Sentinal哨兵模式。

Sentinal就是高可用，master宕机之后，会将slave提升为master继续提供服务。

哨兵组件的作用：
- 集群监控：负责监控 Redis master 和 slave 进程是否正常工作。
- 消息通知：如果某个 Redis 实例有故障，那么哨兵负责发送消息作为报警通知给管理员。
- 故障转移：如果 master node 挂掉了，会自动转移到 slave node 上。
- 配置中心：如果故障转移发生了，通知 client 客户端新的 master 地址。

### 13. 缓存雪崩和缓存穿透

（1）缓存雪崩：如果大量缓存同时失效，导致流量打到数据库，会给数据库造成压力。

为了避免流量全部打到数据库，解决办法可以参考以下几种方式：

 给过期时间加一个随机值。使过期时间随机一些。比如1-3分钟的过期时间。
如果是redis实例挂了的话，可以采用限流或者服务降级。针对非核心业务的话，直接降级，等服务恢复。针对核心业务，服务不能停的话，采用限流的方式，每1w次请求，只允许1000个请求通过。可以采用集群的方式，避免实例不可用的情况


（2）缓存穿透：就是一直访问不存在的key，绕过缓存。这种情况的解决办法就是使用布隆过滤器。

每次设置一个空缓存。这个值可以和业务方沟通好。

使用布隆过滤器。若是缓存中，可以知道数据是否存在就可以直接响应返回。若缓存缺失的话，就去数据库中读取。

前端可以做一些简单的检测，如果是恶意请求，直接过滤掉，不向后端发请求。


（3）缓存击穿：是一个热点的key，在这个key失效的瞬间，持续的高并发就突破了缓存，给数据库造成压力。好想在一个桶上面凿了一个洞。

可以将热点数据设置为永远不过期；

### 14. redis删除方式

惰性删除。惰性删除，避免对每个键，维护删除机制，查询时过期的健值，返回空。无法删除过期但不删除的健
定期删除。定时删除，10s运行一次，快慢模式删除