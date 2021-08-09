# O-Cache

### 关于O-Cache

O-Cache是一个分布式缓存，它在一定程度上是[groupcache](https://github.com/golang/groupcache)的简化版实现。
O-Cache的特性有

- LRU-K缓存策略
- 单机缓存和基于 HTTP 的分布式缓存
- 使用 Go 锁机制防止缓存击穿
- 使用一致性哈希选择节点，实现负载均衡
- 使用 protobuf 优化节点间二进制通信




### 为什么采用LRU-K缓存策略 (package lru)

在LRU缓存策略之上，[LRU-K](https://cloud.tencent.com/developer/article/1664672)可以在一定程度上解决LRU算法“[缓存污染](https://segmentfault.com/a/1190000018810255)”的问题，其核心思想是将“最近使用过1次”的判断标准扩展为“最近使用过K次”：相比于LRU-1，缓存数据更不容易被替换，而且偶发性的数据不易被缓存。在保证了缓存数据纯净的同时还提高了热点数据命中率。

具体实现需要记录两个东西

- cache上的元素
- history上的元素，针对history上的元素，只有其次数超过K时才将其移动到cache上



### 单机缓存 (cache.go, byteview.go)

我们使用 `sync.Mutex` 封装 LRU-K的几个方法，使之支持并发的读写。

我们抽象了一个只读数据结构 `ByteView` 用来表示缓存值，是 O-Cache 主要的数据结构之一。

- ByteView 只有一个数据成员，`b []byte`，b 将会存储真实的缓存值。选择 byte 类型是为了能够支持任意的数据类型的存储，例如字符串、图片等。
- 实现 `Len() int` 方法，我们在 lru.Cache 的实现中，要求被缓存对象必须实现 Value 接口，即 `Len() int` 方法，返回其所占的内存大小。
- `b` 是只读的，使用 `ByteSlice()` 方法返回一个拷贝，防止缓存值被外部程序修改。



### 主体结构 Group (ocache.go)

Group 是 O-Cache 最核心的数据结构，负责与用户的交互，并且控制缓存值存储和获取的流程。

- 一个 Group 可以认为是一个缓存的命名空间，每个 Group 拥有一个唯一的名称 `name`。比如可以创建三个 Group，缓存学生的成绩命名为 scores，缓存学生信息的命名为 info，缓存学生课程的命名为 courses。



### O-Cache HTTP 服务端

分布式缓存需要实现节点间通信，建立基于 HTTP 的通信机制是比较常见和简单的做法。如果一个节点启动了 HTTP 服务，那么这个节点就可以被其他节点访问。今天我们就为单机节点搭建 HTTP Server。



### 一致性哈希

#### 1 为什么使用一致性哈希

对于分布式缓存来说，当一个节点接收到请求，如果该节点并没有存储缓存值，那么它面临的难题是，从谁那获取数据？自己，还是节点1, 2, 3, 4… 。假设包括自己在内一共有 10 个节点，当一个节点接收到请求时，随机选择一个节点，由该节点从数据源获取数据。这样做，一是缓存效率低，二是各个节点上存储着相同的数据，浪费了大量的存储空间。

普通的哈希算法可以，但假如节点丢失就可能造成缓存雪崩。

一致性哈希算法，在新增/删除节点时，只需要重新定位该节点附近的一小部分数据，而不需要重新定位所有的节点，这就解决了上述的问题。

#### 2 数据倾斜问题

如果服务器的节点过少，容易引起 key 的倾斜。为了解决这个问题，引入了虚拟节点的概念，一个真实节点对应多个虚拟节点。假设 1 个真实节点对应 3 个虚拟节点，那么 peer1 对应的虚拟节点是 peer1-1、 peer1-2、 peer1-3（通常以添加编号的方式实现），其余节点也以相同的方式操作。

- 第一步，计算虚拟节点的 Hash 值，放置在环上。
- 第二步，计算 key 的 Hash 值，在环上顺时针寻找到应选取的虚拟节点，例如是 peer2-1，那么就对应真实节点 peer2。

虚拟节点扩充了节点的数量，解决了节点较少的情况下数据容易倾斜的问题。而且代价非常小，只需要增加一个字典(map)维护真实节点与虚拟节点的映射关系即可。

默认哈希函数模为 `crc32.ChecksumIEEE` 算法。



### 缓存击穿

使用 singleflight 防止缓存击穿

> **缓存雪崩**：缓存在同一时刻全部失效，造成瞬时DB请求量大、压力骤增，引起雪崩。缓存雪崩通常因为缓存服务器宕机、缓存的 key 设置了相同的过期时间等引起。

> **缓存击穿**：一个存在的key，在缓存过期的一刻，同时有大量的请求，这些请求都会击穿到 DB ，造成瞬时DB请求量大、压力骤增。

> **缓存穿透**：查询一个不存在的数据，因为不存在则不会写到缓存中，所以每次都会去请求 DB，如果瞬间流量过大，穿透到 DB，导致宕机。

在一瞬间有大量请求get(key)，而且key未被缓存或者未被缓存在当前节点 如果不用singleflight，那么这些请求都会发送远端节点或者从本地数据库读取，会造成远端节点或本地数据库压力猛增。使用singleflight，第一个get(key)请求到来时，singleflight会记录当前key正在被处理，后续的请求只需要等待第一个请求处理完成，取返回值即可。



### Protobuf 通信

#### 1 为什么要使用 protobuf

protobuf 广泛地应用于远程过程调用(RPC) 的二进制传输，使用 protobuf 的目的非常简单，为了获得更高的性能。传输前使用 protobuf 编码，接收方再进行解码，可以显著地降低二进制传输的大小。另外一方面，protobuf 可非常适合传输结构化数据，便于通信字段的扩展。

### 参考：https://geektutu.com/post/geecache.html

