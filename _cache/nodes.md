# Cache的需求

- 需要较高读写性能+命中率
- 支持按写入时间过期
- 支持淘汰策略
- 需要解决gc，大量stw扫描，cpu

![image-20220424221258614](./img/image-20220424221258614.png)



## FastCache

特点:

- 线程安全(多个桶，降低并发冲突)
- 存储大量的cache实体，而且不会被GC扫描（堆内存unix.Mmap）
  - 内存映射的方式可以直接向操作系统申请内存，这块区域不归GC管。所以不管你在这块内存缓存了多少数据，都不会因为GC扫描而影响性能。
  - 但是这里分配的内存再也不会被释放，直到进程重启

- cache空间满了以后，fastcache会自动淘汰老数据块（RingBuf）
- 内存不会预先分配，随用随分配



fastcache为什么快，因为用了这些手段：

1. 使用mmap来成块的分配内存。
   - 每次直接向操作系统要64MB，这些内存都绕开了GC。
   - 每次以64KB为单位请求一个块
   - 在64KB的块内顺序存储，相当于更简单的自己实现的分配算法
2. 整个cache分成512个bucket
   - 相当于有了512个map+512个读写锁，通过这样减少了竞争
   - map类型的key和value都是整形，容量小，且对GC友好
   - 淘汰用轮换的方法+固定次数的set后再清理，解决了（或者说绕开了）碎片的问题



## BigCache

需求如下：

- 支持http协议
- 支持 **10K** RPS (5k 写，5k 读)
- cache对象至少保持10分钟
- 相应时间平均 5ms, p99.9 10毫秒， p99.999 400毫秒
- 其它HTTP的一些需求

为了满足这些需求,要求开发的cache库要保证：

- 即使有百万的缓存对象也要非常快
- 支持大并发访问
- 一定时间后支持剔除



解决GC导致的停顿：

1. 我们知道垃圾回收器检查的是堆上的资源，如果不把对象放在堆上，不就解决这个问题了吗？还真有这样的项目[offheap](https://godoc.org/github.com/glycerine/offheap)，它提供了定制的`Malloc()` 和 `Free()`，但是你的缓存需要基于这些方法定制。当然一些基于垃圾回收的编程语言为了减少垃圾回收的时间，都会提供相应的库，比如[Java: ChronicleMap, Part 1: Go Off-Heap](https://dzone.com/articles/java-chroniclemap-part-1-go-off-heap)。堆外内存很容易产生内存泄漏。
2. 第二种方式是使用[freecache](https://github.com/coocood/freecache)。freecache通过减少指针的数量以零GC开销实现map。它将键和值保存在`ringbuffer`中，并使用索引查找对象。
3. 第三种优化方法是和Go 1.5中一个修复有关([#9477](https://github.com/golang/go/issues/9477)), 这个issue还是描述了包含大量对象的map的垃圾回收时的耗时问题，Go的开发者优化了垃圾回收时对于map的处理，如果map对象中的key和value不包含指针，那么垃圾回收器就会对它们进行优化



缓存的实现离不开如下几种：

1. 原生 `map`
2. `sync.Map`
3. 基于以上二者封装的复合型 `map`

前两者的缺点也很明显：

1. 当 `map` 中存在大量 keys 时，GC 扫描 `map` 产生的停顿将不能忽略（针对 `map` 中存储指针或数据类型底层也是由指针实现这样的场景）
2. 加锁的粒度

提高缓存性能的手段也是明确的：

1. 减少 GC
2. `map` 中尽量避免存储指针
3. 分段（Shards）存储，减少 `lock`

具体实现：

1. 索引（index）与数据（data）分离存储的 `map` 结构
   - 这样 GC 就变成了 `map` 无指针结构 和 `[]byte` 结构的扫描问题
2. GoLang 1.5 版本的 [优化说明](https://github.com/golang/go/issues/9477): 如果 `map` 的 key 或 value 中都不含指针, GC 便会忽略这个 `map`

3. 分片解决并发竞争：bigCache 中使用了分片技术。创建 `N` 个 shard，每个 shard 包含一个带锁的 `cacheShard`，bigCache 将数据分散到不同的 `cacheShard` 进行存储。当从缓存中读写数据时，根据 `HashFunc(key)%N` 选择其中一个 `cacheShard` ，获取缓存锁 `cacheShard.lock`，这样可以大幅降低并发过程中的锁粒度。
4. [ ]byte+map\[uint64\]\[ uint64\]规避GC：从 bigCache 的 `cacheShard` 结构来看，使用了 `map[uint64]uint32` 结构，其中 key 和 value 均无指针结构，其中 value 会追加到一个全局的 `[]byte` 中，每一个 shard 中包含一个全局 `[]byte` 类型的结构 `queue.BytesQueue`。由于此字节切片除了自身对象不包含其他指针，所以 GC 对于整个 `cacheShard` 的标记时间是 `O(1)`
5. 



缺点：

1. 无持久化功能，只能用作单机缓存。
2. bigcache只能等待清理最老的元素的时候把这些"虫洞"删除掉。
3. 在添加一个元素失败后，会清理空间删除最老的元素。
4. 还会专门有一个定时的清理goroutine, 负责移除过期数据。
5. 缓存对象没有读取的时候刷新过期时间的功能，所以放入的缓存对象最终免不了过期的命运。
6. 所有的缓存对象的`lifewindow`都是一样的，比如30分钟、两小时。







参考：

https://pandaychen.github.io/2020/03/03/BIGCACHE-ANALYSIS/

https://colobu.com/2019/11/18/how-is-the-bigcache-is-fast/

https://blog.allegro.tech/2016/03/writing-fast-cache-service-in-go.html

http://liuqh.icu/2021/06/15/go/package/14-bigcache/

https://zhuanlan.zhihu.com/p/487455942





















## FreeCache

特点：

- 能存储数亿条记录（entry） 。
- 零GC开销。
- 高并发线程安全访问。
- 纯Golang代码实现。
- 支持记录（entry）过期。
- 接近LRU的替换算法。
- 严格限制内存的使用。
- 提供一个测试用的服务器，支持一些基本 Redis 命令。
- 支持迭代器。

set操作为什么高效

- 采用二分查找，极大的减少查找entry索引的时间开销。slot切片上的entry索引是根据hash16值有序排列的，对于有序集合，可以采用二分查找算法进行搜索，假设缓存了n个key，那么查找entry索引的时间复杂度为log2(n * 2^-16) = log2(n) - 16。

- 对于key不存在的情况下（找不到entry索引）。
  如果Ringbuf容量充足，则直接将entry追加到环尾，时间复杂度为O(1)。
  如果RingBuf不充足，需要将一些key移除掉，情况会复杂点，后面会单独讲解这块逻辑，不过freecache通过一定的措施，保证了移除数据的时间复杂度为O(1)，所以对于RingBuf不充足时，entry追加操作的时间复杂度也O(1)。

- 对于已经存在的key（找到entry索引）。
  如果原来给entry的value预留的容量充足的话，则直接更新原来entry的头部和value，时间复杂度为O(1)。
  如果原来给entry的value预留的容量不足的话，freecache为了避免移动底层数组数据，不直接对原来的entry进行扩容，而是将原来的entry标记为删除（懒删除），然后在环形缓冲区RingBuf的环尾追加新的entry，时间复杂度为O(1)。





参考：

https://blog.csdn.net/chizhenlian/article/details/108435024

https://www.cxybb.com/article/baidu_32452525/118199343

















fghibbbbccccddde

fghibbbbc-----------

fghibbbbc---e------

fghibbbbc---efgh

ighibbbbc---efgh

fghibbbbccccedde

fghibbbbccccefgh

ighibbbbccccefgh

















