[toc]

# NoSQL
## 为什么要用 NoSQL

一开始，一个基本的网站的访问量小，单个数据库足够。使用静态的网页 html，服务器压力很小。这种情况下：

1. 如果数据量太大，一个机器肯定放不下
2. 超过 300 万条数据，就一定要建立索引。如果数据的索引太大，一个机器内存也放不下这么多索引
3. 如果读写混合（也就是都在一个数据库内），访问量大的时候一个服务器承受不了

如果出现以上三种情况，就必须使用读写分离，然后使用缓存：

- 把数据库分为专门写内容的和读内容的数据库。因为一个网站 80% 的情况都在读，所以读取内容的数据库必须加个缓存来缓解压力

继续发展下去，就变成了「分库分表」+「水平拆分」+ 「MySQL 集群」，用于解决写的压力。

如果要存放定位信息，热门榜单，音乐，这种数据量很多，变化很快的数据，就不能用关系型数据库了。

像用户的个人信息，地理位置，用户自己产生的数据，用户的日志等等爆发式的增长，必须使用 NoSQL 才能很好地处理以上的情况。

---

对于架构师，没有什么是加一层解决不了的：

1. 商品的基本信息
	- 名称、价格、商家信息
	- 使用关系型数据库就可以解决了
2. 商品的描述
	- 评论等文字比较多的数据
	- 使用 MongoDB 这样的文档型数据库中
3. 图片数据
    - 使用分布式文件系统
    - 基础的 FastDFS
    - Hadoop 的 HDFS
4. 搜索引擎
    - ElasticSearch
    - ISearch（阿里巴巴）
5. 商品热门的波段信息
    - 内存数据库
    - Redis、Tair、memcache
6. 商品的交易、外部的支付接口
    - 三方应用

## 什么是 NoSQL

关系型数据库：RDMBS 表格，行，列（POI）

非关系型数据库：NoSQL / Not Only SQL，

很多数据类型的存储不需要固定的格式，不需要多余的操作就可以横向扩展。使用 Map<String, Obeject> 键值对的数据结构就很好。

NoSQL 特点：

1. 数据之间没有关系，方便扩展。
2. 大数据量的高性能。是一种细粒度的缓存。
3. 数据类型多样，不需要事先设计数据库，随取随用。
4. 没有固定的查询语言。
5. 键值对存储，列存储，文档存储，图形数据库（社交关系）
6. 只需保证最终一致性
7. CAP 定理和 BASE（异地多活）

## NoSQL 的四大分类

**KV 键值对**：

- Redis
- Tair
- MemCache

**文档型数据库（BSON 格式，类似于 JSON）**：

- MongoDB：
    - 基于分布式文件存储的数据库，主要用来处理大量的文档
    - 是一个介于关系型数据库和非关系型数据库的中间产品
    - 是非关系型数据库中功能最丰富，最像关系型数据库的产品
- ConthDB

**列存储数据库**：

- HBase
- 分布式文件系统

**图关系数据库**：

- 不是存储图片，而是存储关系的
- 比如：社交拓扑、社交网络、广告推荐
- Neo4j、InfoGrid

# Redis 基础

## Redis 概述

**Re**mote **Di**ctionary **S**erver / 远程字典服务

作用：

1. 内存存储、持久化：内存是“断电即失”，所以要使用持久化（RDB 和 AOF）
2. 效率高，用于高速缓存
3. 发布订阅系统
4. 地图信息分析
5. 计时器、计数器、浏览量

特性：

1. 开源
2. 多样的数据类型
3. 持久化
4. 集群
5. 事务

## Redis 相关知识

Redis 默认有 16 个数据库：

- 默认使用第 0 个数据库
- 可以使用 `select 5` 来切换到第 6 个数据库（从 0 到 15，共 16 个）
- 可以使用 `dbsize` 来查看当前数据库的大小（存储了多少数据）
- 可以使用 `flushdb` 来清理当前数据库，`flushall` 清空全部的数据库

Redis 是单线程的：

- Redis 基于内存操作，它的性能基于机器的内存和网络带宽
- CPU 对 Redis 没什么影响，所以能用单线程的话，没必要使用多线程（反正也提升不了性能）

为什么 Redis 使用单线程还那么快？

- 高性能的服务器 *不一定* 是多线程的
- 多线程 *不一定* 比单线程效率高，因为 CPU 有上下文切换，会消耗资源
- 核心：
    - Redis 将数据存放在内存中
    - 对于内存系统来说，没有上下文切换的时候，效率最高
    - 而如果使用了多线程，CPU 就会上下文切换，消耗资源
    - 只要多次读写都在同一个 CPU 上，单线程操作内存的效率就是最高的

## Redis-Key 的基本命令

`keys *` ：查看所有 key

`exists [key]`：返回 0，说明没有这个 key；返回其他数字说明有这个 key

`set [key] [value]`：存放 key 和 value

`get [key]`：取出 key 的 value

`del [key]`：删除 key

`move [key] [数据库名称]`：将 key 移动/剪切到其他数据库中（数据库从 0 开始计算，移动成功后，当前数据库内的相应 key 会被删除）

`expire [key] [多少秒]`：设置这个 key 在多少秒后过期

`type [key]`：查看该 key 的类型

# Redis 的类型

## String 字符串类型

这里查看的是 key 为 string 字符串的情况，value 除了字符串，还可以是数字等。

使用场景可以是：

- 计数器 `incr [key]`
- 存储对象
- 比如统计 uid 为 1234 的用户的粉丝数，每多一个粉丝就加 1:`incr uid:1234:follewers`

首先看一下操作字符的相关操作：

`set [key] [value]` 后，还可以直接在 value 后面添加字符串，也就是 append。

这里设置 key1 为 abc，然后在 abc 后面再使用 append 命令添加 def。

注意：*如果当前 key 不存在，那么 `append [key] [追加的字符串]` 就会生成一个新的 key，存储的 value 为「追加的字符串」*，相当于 set key 命令。

返回的「(integer) 6」代表该 value 的新长度。

要得出该字符串长度还可以使用 `strlen [key]` 命令。

得出的新 value 为 abcdef：

```bash
127.0.0.1:6379> set key1 abc
OK
127.0.0.1:6379> append key1 def
(integer) 6
127.0.0.1:6379> get key1
"abcdef"
127.0.0.1:6379> strlen key1
(integer) 6
```

---

关于 redis 内的自增操作：

在很多时候会有「阅读量」「浏览量」的数据，我们可以：

`set views 0`：新建作为存储「阅读量」的 key，先设置为 0

此时 `get views` 还是会返回默认的 value 值，即 0

如果有一个用户查看了该文章，我们可以触发命令 `incr views`，那么，key 为 views 的value 就会从默认的 0，变为 0 + 1，即 1

此时 `get views` 变为 1

也就是说，`incr [key]` 命令可以让该 key 里面的 value 自动加 1，相当于 Java 的 `i++`

如果需要减少 1，那么使用 `decr [key]`，该 key 的 value 就减少了 1，相当于 Java 的 `i--`

如果要一次性增加多个数值，那么 `incrby [key] [增加多少]`，比如 `incrby views 10`，就会让 key 为 views 的 value 一次性加 10

一次性减少的数值就使用 `decrby [key] [增加多少]`

---

获取部分字符串：

相当于 Java 里面的 substring 方法，把一个字符串的其中一个部分拿出来。

`getrange [key] [startIndex] [endIndex]`：获取 key 的 value，只出取出该 value 从 startIndex 开始（注意是 index）到 endIndex 范围的字符串值。

```bash
127.0.0.1:6379> set num 0123456789
OK
127.0.0.1:6379> getrange num 0 0
"0"
127.0.0.1:6379> getrange num 0 1
"01"
127.0.0.1:6379> getrange num 4 9
"456789"
```

如果是 `getrange [key] [startIndex] -1`，也就是最后一个参数 endIndex 为 -1，说明 endIndex 为字符串最后的位置。

也就是说，如果是 0 到 -1，就是取位置为 0 的字符串到最后 1 个字符串，也就是获取整个字符串。

如果是取 1 到 -1，就是从位置为 1 开始，一直取到最后一个位置的字符串：

```bash
127.0.0.1:6379> getrange num 0 -1
"0123456789"
127.0.0.1:6379> getrange num 1 -1
"123456789"
```

---

还可以使用 `setrange [key] [开始替换的位置] [需要替换的字符]` 来替换某个位置开始的字符。

假设 key 为 num，value 为 0123

如果使用 `setrange num 1 ab`，就说明要把 key 为 num 的 value，从 index 为 1 的位置（从 0 开始），替换字符为 ab。（替换的 string 长度为多少，就能替换多少）

最后这个 key 为 num 的 value 就变为 0ab3

再看一个例子加深印象：

```bash
127.0.0.1:6379> set key2 abcdefg
OK
127.0.0.1:6379> setrange key2 0 xxx
(integer) 7
127.0.0.1:6379> get key2
"xxxdefg"
```

---

存储 key 的时候，一并设置过期时间，使用 `setex [key] [过期时间] [value]` 来设置

比如设置 `setex key3 30 "30 seconds to expire"` 代表让 key3 的过期时间为 30 秒。

在 30 秒之内，我们可以通过 `ttl key3` 来查看 key3 还会多久过期。

等 30 秒过期后，我们通过 `ttl key3` 命令，它会返回 -2，说明 key3 已经过期了。这个时候我们 `get key3` 会返回 (nil) 代表没有这个 key

---

在分布式锁中，我们存入 key 还要判断这个 key 之前是否存在。

使用 `setnx [key] [value]`，如果这个 key 并不存在，那么它就能存储成功，并返回 1

如果这个 key 之前已经存在，那么它就不会创建，并会返回 0，告诉你这个 key 已经在 redis 数据库中了。

这样的话，我们就可以加上乐观锁。

---

批量设置和读取 key：

批量设置 key 和 value：`mset [第一个key] [第一个value] [第二个key] [第二个value]...`

比如：`mset k1 v1 k2 v2 k3 v3`

注意，使用 `mset` 的时候，如果 key 已经存在，那么它会 **覆盖原先的 key 里面的 value**，也就是说，如果它会改变那个 key 原先的 value，放入新的 value

如果不想覆盖原来的 key，那么就要判断是否存在这个 key，使用 `setnx` 命令就可以了。

这里假设原先有一个 key 为 k1，但是没有 key 为 k2：

```bash
msetnx k1 v1 k2 v2
```

此时，redis 发现虽然 k2 不存在，但是 k1 是存在的，所以存入失败，返回 0

注意，就算之前 k2 不存在，此时也没办法存入，因为 k1 已经存在了，也就是说 **msetnx 是原子性的（mset 也是原子性的）**，要么一起成功，要么一起失败。

如果要批量读取 key 里面的 value：`mget [第一个key] [第二个key]...`

比如：

```bash
127.0.0.1:6379> mget k1 k2 k3
1) "v1"
2) "v2"
3) "v3"
```

---

存入对象（JSON 格式）：

假设要存入对象 user:1 为 json 对象，一般来说这样设置：

```
set user:1 {name:john,age:16}
```

高级做法是：

```
mset user:1:name john user:1:age 16
```

也就是说，在 Redis 中，key 可以设置为 `对象:对象的id:对象的属性` 这样的形式。

实际应用中，可以 `set article:5:views 0` ，表示设置编号为 5 的文章（article）的阅读量（views）为 0，等阅读量每次增加 1 个的时候，再 `incr article:5:views` 。

这样就可以实现 key 的复用，方便操作这种经常要用到的 key

---

先 get 再 set 的操作。

假设没有 key 为 k4：

```bash
127.0.0.1:6379> getset k4 v4
(nil)
127.0.0.1:6379> get k4
"v4"
127.0.0.1:6379> getset k4 new
"v4"
127.0.0.1:6379> get k4
"new"
```

也就是说，使用了 `getset [key] [val]` 的话，会首先返回原先 key 的值（原先没有这个 key 的话，返回「(nil)」），然后再更改该 key 的 value 为新设置的 value。

这是符合 **CAS**，也就是对比再交换原理的操作，在实际应用中，可以用来更新数值，也就是做 update

## List 列表类型

在 Redis 里面，List 类型本质上是一个「双向链表」：

- 它的 left node 和 right node 都可以插入值
- 如果 key 不存在，就会创建新的链表
- 如果 key 存在，就会新增内容
- 如果移除了所有值，就会变成空链表，也就是不存在这个 list
- 在两边插入和改动值的时候，效率最高；操作中间元素效率会低一点

我们可以使用 List 类型实现栈、消息队列、阻塞队列等。

**注意：Redis 中 List 类型的 key 是可以有重复的 value 的**。比如 key 名称为 allone 的列表中，value 可以有「1,2,3」，也可以有「1,1,1」，也就是说 allone 列表可以重复多个 1。

记住，操作 List 的时候，命令行是「L」开头的话，就是从左边放入。如果是「R」开头，就从列表的右边放入。

也就是说 `lpush [key] [val01]` 是从列表的左边插入 val01，如果继续 `lpush [key] [val02]`，那么 val02 就会从左边插入，然后把 val01 挤到右边，从而变成 val02，val01 的顺序。

此时如果 `rpush [key] [val03]`，那么整个列表的从左往右的顺序就是 val02，val01，val03。

如果要查看列表，只能按照从左往右的顺序查，指令为 `lrange [key] [startIndex] [endIndex]`，其中 endIndex 为 -1 时，代表整个列表的最后一个位置。

删除命令也区分左和右，分别为 `lpop [key]` 和 `rpop [key]`

简单的操作如下：

```bash
127.0.0.1:6379> lpush numList one
(integer) 1
127.0.0.1:6379> lpush numList two
(integer) 2
127.0.0.1:6379> lpush numList three
(integer) 3
127.0.0.1:6379> lrange numList 0 -1
1) "three"
2) "two"
3) "one"
127.0.0.1:6379> rpush numList four
(integer) 4
127.0.0.1:6379> lrange numList 0 -1
1) "three"
2) "two"
3) "one"
4) "four"
127.0.0.1:6379> lpop numList
"three"
127.0.0.1:6379> lpop numList
"two"
127.0.0.1:6379> rpop numList
"four"
127.0.0.1:6379> lrange numList 0 -1
1) "one"
```

`llen [key]` 表示 **L**ist **Len**gth，返回名称为 key 的列表的长度，也就是存放了多少个 value

---

如果要获取固定的值的话，就要通过 `lindex` 命令来获取该 index 位置的 value：

```bash
127.0.0.1:6379> rpush mylist 0 1 2 3 4 5
(integer) 6
127.0.0.1:6379> lrange mylist 0 -1
1) "0"
2) "1"
3) "2"
4) "3"
5) "4"
6) "5"
127.0.0.1:6379> lindex mylist 1
"1"
127.0.0.1:6379> lindex mylist 0
"0"
127.0.0.1:6379> lindex mylist 4
"4"
127.0.0.1:6379>
```

---

移除某一个特定的值，可以使用 `lrem [key] [相同名称的指定val的数量] [需要移除的指定val]`

为了举例，先创建一个 List：

```bash
127.0.0.1:6379> rpush abc a b c a c b
(integer) 6
127.0.0.1:6379> lrange abc 0 -1
1) "a"
2) "b"
3) "c"
4) "a"
5) "c"
6) "b"
```

此时，要**移除从左到右数的第 1 个 a**的话，就使用 `lrem abc 1 a`，代表「从 **L**eft/左边开始 **rem**ove/移除 key 为 **abc** 的列表内的 **1** 个元素，该元素的名称为 **a**」。此时列表为「b,c,a,c,b」，也就是只删除了第 1 个 a，第 2 个 a 没有变化。

同理：

```bash
127.0.0.1:6379> lrem abc 2 b
(integer) 2
127.0.0.1:6379> lrange abc 0 -1
1) "c"
2) "a"
3) "c"
```

此时，abc 列表中只有 2 个元素的名称为 c，如果我们要求移除 3 个 c 的话，它也可以执行，只不过只会删除掉 2 个 c 而已：

```
127.0.0.1:6379> lrem abc 3 c
(integer) 2
127.0.0.1:6379> lrange abc 0 -1
1) "a"
```

如果执行的命令是 `lrem abc -1 a`，也就是移除 abc 列表中 -1 个 a 元素的话，它会把 -1 理解成 1，所以只会删除一个元素 a

---

如果要截取列表中从 startIndex 开始（包括 startIndex），到 endIndex 结束（包括 endIndex）的元素，那么可以使用 `ltrim [key] [startIndex] [endIndex]`：

```bash
127.0.0.1:6379> rpush mylist zero one two three four five
(integer) 6
127.0.0.1:6379> lrange mylist 0 -1
1) "zero"
2) "one"
3) "two"
4) "three"
5) "four"
6) "five"
127.0.0.1:6379> ltrim mylist 1 3
OK
127.0.0.1:6379> lrange mylist 0 -1
1) "one"
2) "two"
3) "three"
```

---

如果要移除列表最后一个元素，并将该元素移动到新的列表中：

可以使用 `rpoplpush [原列表key] [新列表key]`，代表「先 rpop 原列表」，也就是先移出原列表最右边的元素，然后再「lpush」到新列表中。

```bash
127.0.0.1:6379> rpush origin one two three
(integer) 3
127.0.0.1:6379> lrange origin 0 -1
1) "one"
2) "two"
3) "three"
127.0.0.1:6379> rpoplpush origin dest
"three"
127.0.0.1:6379> lrange origin 0 -1
1) "one"
2) "two"
127.0.0.1:6379> lrange dest 0 -1
1) "three"
```

这里示范的是在 origin 列表中的最后一个元素，被添加到新的 dest 列表中。

---

如果要将列表中，指定 index/下标位置的值，替换为另一个值，可以使用：

`lset [key] [index] [替换为这个值]`

这里要注意分清楚情况，如果原先没有这个 key，也就是列表不存在，那么 `lset` 命令肯定无法执行：

```bash
127.0.0.1:6379> exists mylist
(integer) 0
127.0.0.1:6379> lset mylist 0 myvalue
(error) ERR no such key
```

就算列表里面有值，如果替换的时候超过了列表的大小，也会无法执行：

```bash
127.0.0.1:6379> lpush mylist myvalue
(integer) 1
127.0.0.1:6379> lset mylist 1 newvalue
(error) ERR index out of range
```

这里创建一个列表，存放 3 个元素，然后替换 index 为 1，也就是第 2 个元素：

```bash
127.0.0.1:6379> rpush keylist v1 v2 v3
(integer) 3
127.0.0.1:6379> lrange keylist 0 -1
1) "v1"
2) "v2"
3) "v3"
127.0.0.1:6379> lset keylist 1 two
OK
127.0.0.1:6379> lrange keylist 0 -1
1) "v1"
2) "two"
3) "v3"
```

如果想要替换最后一个元素，却不知道最后一个元素的 index 的话，可以指定 index 为 -1：

```bash
127.0.0.1:6379> lset keylist -1 last
OK
127.0.0.1:6379> lrange keylist 0 -1
1) "v1"
2) "two"
3) "last"
```

---

如果要在列表（key）中，在某个具体的*前面*或*后面*插入一个新的 value：

使用 `linsert [key] [before或者after] [在哪个value的前后] [插入的值]`

```bash
127.0.0.1:6379> rpush mylist v1 v2 v3 v4
(integer) 4
127.0.0.1:6379> linsert mylist before v2 b4v2
(integer) 5
127.0.0.1:6379> lrange mylist 0 -1
1) "v1"
2) "b4v2"
3) "v2"
4) "v3"
5) "v4"
127.0.0.1:6379> linsert mylist after v2 afterv2
(integer) 6
127.0.0.1:6379> lrange mylist 0 -1
1) "v1"
2) "b4v2"
3) "v2"
4) "afterv2"
5) "v3"
6) "v4"
```
**注意！如果列表中有重复的元素，那么这个插入操作只会以最左边的第 1 个该元素（也就是 index 最小的那个）为基准，其余重复元素无法作为基准**。

## Set 集合类型

Set 中的元素是无序，且不能重复的，也就是**无序不重复集合**


存储 set 集合的元素使用 `sadd [key] [val]`，查看该 set 集合的所有 member/元素使用 `smembers [key]`（注意 member 后面有 s），查看某个元素是否（is）在该 set 集合中的元素（member）使用 `sismember [set集合/key] [需要查找的元素/val]`

```bash
127.0.0.1:6379> sadd myset one two three
(integer) 3
127.0.0.1:6379> smembers myset
1) "three"
2) "two"
3) "one"
127.0.0.1:6379> sismember myset four
(integer) 0
127.0.0.1:6379> sismember myset three
(integer) 1
```

如果要获取 set 集合的大小，也就是有多少个元素，使用：`scard [key]`，它会返回个数。

如果要**移除某个具体的元素**，使用：

`srem [key] [要删除的val]` 。

如果要**移除某个随机的元素**，根据 set 的无序的特性，我们随便 pop 出来一个元素，都是随机移除该 set 集合里面的元素：

命令为 `spop [key] [移除多少个]` ，其中的「移除多少个」如果不填的话，默认为 1，也就是只随机删除 1 个元素。

---

因为 redis 中的 set 是无序不重复的集合，所以我们可以实现<u>随机抽取</u>的操作（随机移除就是 spop 指令）：

`srandmember [key] [抽取多少个]`

比如我想抽取 key 为 myset 的 set 集合中，2 个元素，就使用
`srandmember myset 2`，它会返回随机 2 个元素。

如果要抽取的数量，大于或等于 set 集合元素的个数，那么它会返回该 set 集合内的所有元素。

---

如果要将某个 set 集合中的元素，先移除，然后再移动到另一个 set 集合中的话：

`smove [原来的key] [移动到的key] [需要移动的value]`

---

在社交媒体中，有一个「共同关注」的功能，在数学中叫做「交集」，也可以使用 set 集合来实现。

只要是数学中的合集，像「差集」「交集」和「并集」这类集合，都可以使用 set 来完成。

这里先设计两个 key，分别为 k1 和 k2。其中 k1 存放 a、b、c；k2 存放 c、d、e，也就是说，共同的交集只有元素 c

```bash
127.0.0.1:6379> sadd k1 a b c
(integer) 3
127.0.0.1:6379> sadd k2 c d e
(integer) 3
```

我们要找到 k1 和 k2 之间的**差集**，也就是两个 set 集合中，没有交集的部分，使用 `sdiff [第一个key] [第二个key] [第三个key] ...`：

```bash
127.0.0.1:6379> sdiff k1 k2
1) "a"
2) "b"
```

如果要着**交集**，就用 `sinter [第一个key] [第二个key] [第三个key] ...`：

```bash
127.0.0.1:6379> sinter k1 k2
1) "c"
```

如果要取**并集**，也就是两个集合中的所有元素，使用 `sunion [第一个key] [第二个key] [第三个key] ...`：

```bash
127.0.0.1:6379> sunion k1 k2
1) "a"
2) "b"
3) "c"
4) "e"
5) "d"
```

在实际应用中，可以将用户所有关注的人放在一个 set 集合中，将所有粉丝也放在一个集合中，或者其他信息放在集合中，来实现：

- 共同关注
- 共同爱好
- 推荐好友

## Hash 哈希类型

相当于 Map 集合。key 对应的 value 是 Map，这个 Map 又是由 key 和 value 的键值对组成的。

可以使用 `hset [key] [field] [value]` 来设置，其中 [key] 代表这个 HashMap 的 key，field 代表这个 HashMap 里面的其中一个 key，value 代表这个 HashMap 里面的其中一个 value。也可以使用 `hmset` 来批量赋值。

使用 `hget [key] [field]` 来获取这个 key 代表的 HashMap 里面的 key 的 value

比如有一个 key 叫做 k1。在 k1 内有键值对作为 value：{f1: v1}，所以相互关系就变成了「k1:{f1:v1}」。如果再在 k1 内添加新的键值对，就变为了「k1:{f1:v1,f2:v2,f3:v3}」：

```bash
127.0.0.1:6379> hmset k1 f1 v1 f2 v2 f3 v3
OK
127.0.0.1:6379> keys *
1) "k1"
127.0.0.1:6379> hmget k1 f1 f2 f3
1) "v1"
2) "v2"
3) "v3"
```

如果要获取某个 key 里面所有的 HashMap 键值对，使用 `hgetall [key]` 命令：

```bash
127.0.0.1:6379> hgetall k1
1) "f1"
2) "v1"
3) "f2"
4) "v2"
5) "f3"
6) "v3"
```

如果只想获取某个 key 为名称的 HashMap 里面的所有的 key（field/字段），使用 `hkeys [key]`。

如果要获取的是 HashMap 内所有的 value，使用 `hvals [key]`

如果要删除 HashMap 的值，使用 `del [key] [field]...`，比如，我要删除 k1 为 key 的 HashMap 里面的 f1 和 f2 为名称的 key 及其 value：

```bash
127.0.0.1:6379> hdel k1 f1 f2
(integer) 2
127.0.0.1:6379> hgetall k1
1) "f3"
2) "v3"
```

如果要查看 HashMap 里**有多少对**键值对（一对键值对 = 1 个 key + 1 个 value），使用 `hlen [key]`。

判断名称为 key 的 HashMap 中，是否存某个 key（field），使用 `hexists [key] [field]` 。

先判断是否存在，再赋值的操作可以使用 `hsetnx [key] [field] [value] ...`，如果有一个已经存在，那么这条指令全部都会失效。

---

Hash 类型非常类似 String 类型，所以自增（自减）操作也是可以实现的，不过这里要指定自增长了多少。

比如设置一个 key 为 num 的 HashMap，其中的 field 名称为 views，将 views 的初始值设定为 5。然后使用 `hincrby [key] [field] [增加多少]` 的指令让它增加 10：

```bash
127.0.0.1:6379> hset num views 5
(integer) 1
127.0.0.1:6379> hincrby num views 10
(integer) 15
127.0.0.1:6379> hget num views
"15"
```

因为没有自带「自减操作」，所以可以使用 `hincrby [key] [field] [增加负数/使用负数相当于减少]` 来实现减少的操作。

---

在实际应用中，Hash 类型可以用来操作经常变更的数据，尤其是用户信息等经常变动的数据。

也就是说，**Hash 比 String 更适合对象的存储**。

```bash
127.0.0.1:6379> hmset user:1 name john age 16 gender male
OK
127.0.0.1:6379> hget user:1 name
"john"
127.0.0.1:6379> hget user:1 gender
"male"
```

## Zset 有序集合类型

有序集合的底层数据结构是跳跃链表。 

实际上就是在 Set 的基础上增加了一个用于排序的值（score）。

实际应用的时候，可以用来存储「排序表」，还有「普通消息」和「重要消息」的权重判断，热门排行榜也可以用这个来实现。

使用的方法为 `zadd [key] [用于排序的score] [value]...` 查看的时候，使用 `zrange [key] [startIndex] [endIndex]`：

```bash
127.0.0.1:6379> zadd num 0 zero
(integer) 1
127.0.0.1:6379> zadd num 2 two
(integer) 1
127.0.0.1:6379> zadd num 1 one
(integer) 1
127.0.0.1:6379> zrange num 0 -1
1) "zero"
2) "one"
3) "two"
127.0.0.1:6379> zadd num 5 five 3 three 4 four
(integer) 3
127.0.0.1:6379> zrange num 0 -1
1) "zero"
2) "one"
3) "two"
4) "three"
5) "four"
6) "five"
```

如果想筛选在 score 的区间内的 value，可以使用 `zrangebyscore [key] [最小的score] [最大的score]` （闭合区间，包括最小和最大的 score）

注意！**-inf** 表示无穷小，**+inf** 表示无穷大，也就是可以根据这两个来取全部的值。

如果在筛选后还想带上具体的 scores，就在后面加上 `withscores` 关键字：

```bash
127.0.0.1:6379> zadd salary 5000 john
(integer) 1
127.0.0.1:6379> zadd salary 6000 sally
(integer) 1
127.0.0.1:6379> zadd salary 4000 harry
(integer) 1
127.0.0.1:6379> zrangebyscore salary -inf +inf
1) "harry"
2) "john"
3) "sally"
127.0.0.1:6379> zrangebyscore salary 3000 5000
1) "harry"
2) "john"
127.0.0.1:6379> zrangebyscore salary -inf +inf withscores
1) "harry"
2) "4000"
3) "john"
4) "5000"
5) "sally"
6) "6000"
```

如果想**从大到小**给 zset/sorted set 排序（也就是获取 set 所有的值），那么就使用 `zrevrange [key] [startIndex] [endIndex]`，这个也可以使用 `withscores` 关键字：

```bash
127.0.0.1:6379> zrevrange salary 0 -1
1) "sally"
2) "john"
3) "harry"
127.0.0.1:6379> zrevrange salary 0 -1 withscores
1) "sally"
2) "6000"
3) "john"
4) "5000"
5) "harry"
6) "4000"
127.0.0.1:6379> zrevrange salary 0 1 withscores
1) "sally"
2) "6000"
3) "john"
4) "5000"
```

**从小到大**就是普通的 `zrange [startIndex] [endIndex]` 即可。

移除 zset 中的指定元素，使用 `zrem [key] [移除的元素] ...`，这里移除 sally 和 harry，只留下 john：

```bash
127.0.0.1:6379> zrem salary sally harry
(integer) 2
127.0.0.1:6379> zrange salary 0 -1
1) "john"
```

获取有序集合 zset 中 value 的个数：`zcard [key]`。


## Geospatial 地理位置信息

> GEO 的底层实现原理是 zset，我们可以使用 zset 的命令来操作 GEO

这个功能可以根据 longitude（经度）和 latitude（纬度）来推算地理位置的信息，两地之间的距离，附近位置的人等等，实际应用中就是：朋友的定位，附近的人，打车的位置。

设置一个位置的方法：`geoadd [key/总体区域名称] [经度] [纬度] [该位置名称]...`

其中 key 写成「国家:地区」的格式比较好，注意「地球两极」是无法直接添加的，我们一般也会通过 Java 程序来一次性导入数据：

```bash
127.0.0.1:6379> geoadd china:city 116.23128 40.22077 beijing 121.48941 31.40527 shanghai
(integer) 2
127.0.0.1:6379> geoadd china:city 113.88308 22.55329 shenzhen
(integer) 1
```

获取指定 key 的经纬度信息可以使用 `geopos [key] ...`（pos 代表 position）：

```bash
127.0.0.1:6379> geopos china:city beijing shanghai
1) 1) "116.23128265142440796"
   2) "40.22076905438526495"
2) 1) "121.48941010236740112"
   2) "31.40526993848380499"
```

如果想获取两个位置之间的直线距离，可以使用 `geodist [key] [位置1] [位置2] [单位]`（dist 代表 distance），如果不填 [单位]，默认是「m/米」，还可以选择「mi/英里」和「ft/英尺」：

```bash
127.0.0.1:6379> geodist china:city beijing shenzhen km
"1977.4843"
127.0.0.1:6379> geodist china:city beijing shenzhen m
"1977484.3419"
127.0.0.1:6379> geodist china:city beijing shenzhen
"1977484.3419"
```

---

如果想寻找附近的人，可以使用 `georadius [key] [经度] [纬度] [半径] [单位]` （radius 表示半径）。其中 [key] 就是待查找的所有位置的集合，原点位置就是 [经度] 和 [纬度] 所呈现的位置，[半径]也是以该原点位置为半径。

在名称为 china:city 的 key 中，有 shenzhen、shanghai 和 beijing 的地理数据，现在要查找经度为 110 纬度为 30 的区域内，半径为 1000 km 和 10000 km 的城市：

```bash
127.0.0.1:6379> georadius china:city 110 30 1000 km
1) "shenzhen"
127.0.0.1:6379> georadius china:city 110 30 10000 km
1) "shenzhen"
2) "shanghai"
3) "beijing"
```

如果想带上经纬度信息的话，使用关键字 `withcoord` （coord 表示 coordinate，是坐标的意思）：

```bash
georadius china:city 110 30 10000 km withcoord
1) 1) "shenzhen"
   2) 1) "113.88307839632034302"
      2) "22.55329111565713873"
2) 1) "shanghai"
   2) 1) "121.48941010236740112"
      2) "31.40526993848380499"
3) 1) "beijing"
   2) 1) "116.23128265142440796"
      2) "40.22076905438526495"
```

如果想带上各地与指定区域的直线距离，也就是查出在指定位置的半径内，所有的区域，并附带上该区域与指定位置的直线距离，使用关键字 `withdist`：

```bash
127.0.0.1:6379> georadius china:city 110 30 10000 km withdist
1) 1) "shenzhen"
   2) "914.1294"
2) 1) "shanghai"
   2) "1109.3250"
3) 1) "beijing"
   2) "1269.4837"
```

如果只想查出最近距离的指定个数个位置，可以使用关键字 `count [再带上指定出现多少个]`。比如我只想找到附近的 2 个最近符合条件的城市：

```bash
georadius china:city 110 30 10000 km count 2
1) "shenzhen"
2) "shanghai"
```

而且，`withcoord`、`withdist`和`count [多少]` 关键字可以一起使用

---

`georadius` 的相关命令是根据给定坐标来查询附近的坐标的，如果要根据已经录入的坐标，来查找附近的坐标，可以使用：

`georadiusbymember [key] [key里面存入的坐标的名称] [半径] [单位]`

其中，查找的范围是以 [key里面存入的坐标的名称] 为 [半径]，查找 [key] 里面已经录入的所有坐标，返回的值是符合条件的坐标（包括它自己）：

```bash
127.0.0.1:6379> georadiusbymember china:city beijing 1000 km
1) "beijing"
127.0.0.1:6379> georadiusbymember china:city beijing 1500 km
1) "shanghai"
2) "beijing"
127.0.0.1:6379> georadiusbymember china:city beijing 2000 km
1) "shenzhen"
2) "shanghai"
3) "beijing"
```

这里也可以使用 `withcoord`、`withdist`和`count [多少]` 关键字。

---

使用 zset 的命令来操作 GEO：

```bash
127.0.0.1:6379> zrange china:city 0 -1
1) "shenzhen"
2) "shanghai"
3) "beijing"
127.0.0.1:6379> zrem china:city beijing
(integer) 1
127.0.0.1:6379> zrange china:city 0 -1
1) "shenzhen"
2) "shanghai"
```

## Hyperloglog 基数统计

> 基数（きすう、cardinal number又はcardinal）とは、集合の濃度 (cardinality) （大きさ、サイズ）を測るために定義された自然数の一般化である。

能够一一对应的两个元素，被称为互相对等集合。比如 3 个人和 3 匹马就能够一一对应。

可以这样理解，**基数：所有重复的数据都被算作一个，然后统计共有多少个数据**。比如有「1, 2, 2, 3, 3, 3」这样的数据，其中剔除重复的，就是「1, 2 , 3」也就是有 3 个数据，所以基数就是 3。（3 代表把重复的数据算作 1 个多话，这里有 3 个数据）

注意：统计基数的时候，可以容许有一定的误差。

Hyperloglog 是用来做基数统计的算法：

- 优点是它所占用的内存是固定的：2^64 不同元素的基数，只需要使用 12KB 的内存容量
- 缺点是有 0.81% 的误差
- Philippe Flajolet 是这个算法的提出者，所以命令是以 PF 开头的

实际应用，比如统计网页的 UV（页面访问量）这种可以忽略误差的任务：

- 因为一个人访问一个网站多次，还是算作一次，所以传统的方法就使用 set 集合来存储访问了页面的用户 id，然后统计 set 集合中元素的数量
- 如果使用 set 集合来保存用户 id，数据量大的话就比较麻烦。因为目的是为了计数，而不是保存用户的 id
- 使用 Hyperloglog 算法来统计用户访问量，即便到了 2^64 次，也仅需 12KB 的内存容量

添加数据的时候，使用 `pfadd [key] [元素] ...`。

查看数据的时候，使用 `pfcount [key]` 来查看基数。

如下所示，firstDayViews 存放了点击该网址的用户 ID（从 001 到 005）共 5 位，secondDayViews 存放了 7 位（001 到 007）。其中，同一个用户产生的多次数据只会被计为一次：

```bash
127.0.0.1:6379> pfadd firstDayViews 001 001 002 002 002 003 004 005
(integer) 1
127.0.0.1:6379> pfcount firstDayViews
(integer) 5
127.0.0.1:6379> pfadd secondDayViews 001 002 003 004 005 006 007 007
(integer) 1
127.0.0.1:6379> pfcount secondDayViews
(integer) 7
```

如果要统计 firstDayViews 和 secondDayViews 合并后的基数，可以使用 `pfmerge [合并后key的名称] [需要被合并的key]...` 来先把两个 key 中的数据合并起来，然后再统计：

```bash
127.0.0.1:6379> pfmerge firstAndSecondDayViews firstDayViews secondDayViews
OK
127.0.0.1:6379> pfcount firstAndSecondDayViews
(integer) 7
```

这样的话，就可以统计出，前 2 天有 001 到 007 共 7 名用户多次阅读过该网站，访问量就为 7。

## Bitmap 位图场景

> 位存储：数据内只有 0 和 1，普通的情况下就是 0，特殊的情况下就设定为 1（操作二进制位来进行记录）

只有 2 个状态的场景，都可以使用 Bitmap。比如说：

- 疫情统计，没有得病的人就设定为 0，感染了疫情的人就设定为 1
- 统计用户信息：活跃或不活跃，登陆或未登陆，打卡或未打卡

使用 `setbit [key] [offset] [value]` 来标记即可。这里类似 HashMap，[offset] 和 [value] 就像键值对，同一个 [key] 可以有多个 [offset] 和 [value] 组成的键值对。

比如说，要记录打卡的情况，把打卡操作命名为「sign」，然后使用 0 到 6 的数字，代表一周从星期日到星期六（这就是 [offset]）。最后，当 value 为 0 表示没有打卡，value 为 1 表示打了卡：

```bash
127.0.0.1:6379> setbit sign 0 1
(integer) 0
127.0.0.1:6379> setbit sign 1 1
(integer) 0
127.0.0.1:6379> setbit sign 2 0
(integer) 0
127.0.0.1:6379> setbit sign 3 1
(integer) 0
127.0.0.1:6379> setbit sign 4 1
(integer) 0
127.0.0.1:6379> setbit sign 5 0
(integer) 0
127.0.0.1:6379> setbit sign 6 0
(integer) 0
```

根据上述例子，可以得知，该用户在星期二、星期五和星期六没有打卡，其余时间都打了卡。如果要查看的话，使用 `getbit [key] [offset]` 来查看某一个位置是 0 还是 1：

```bash
127.0.0.1:6379> getbit sign 2
(integer) 0
127.0.0.1:6379> getbit sign 3
(integer) 1
```

上面的例子表示星期二没有打卡，星期三打了卡。如果要统计打了卡的天数，也就是 **value 为 1 的次数**可以使用统计操作指令 `bitcount [key]` 来统计：

```bash
127.0.0.1:6379> bitcount sign
(integer) 4
```

这代表有 4 天 value 为 1，也就是打了卡。

# Redis 事务相关

## Redis 事务

**Redis 的单条命令是保证原子性的，然而 <u>Redis 的事务并不保证原子性</u>！**

**Redis 事务的本质：一组命令的集合。一个事务中的所有命令都会先被序列化，在执行事务的过程中，会按照序列化的顺序来执行。**

Redis 执行一系列的命令/事务的特性：

- 一次性
- 顺序性
- 排他性

**Redis 事务没有隔离级别的概念！所有的命令在事务中，并不是直接执行的，而是要先放入队列集合中，只有我们发起执行命令 `exec` 的时候，才会被执行。**

Redis 事务的正常执行顺序：

1. 开启事务：`multi`
2. 命令入队：输入需要被执行的命令
3. 执行事务：`exec`

使用 `multi` 命令开启事务后，我们按照顺序，输入要被执行的命令，此时命令进入了队列，并没有被执行。等我们发送 `exec` 命令后，它会一次性执行队列中存储的命令，然后返回每一个命令的结果。

```bash
127.0.0.1:6379> multi
OK
127.0.0.1:6379> set k1 v1
QUEUED
127.0.0.1:6379> set k2 v2
QUEUED
127.0.0.1:6379> get k2
QUEUED
127.0.0.1:6379> set k3 v3
QUEUED
127.0.0.1:6379> exec
1) OK
2) OK
3) "v2"
4) OK
```

如果中途要**放弃事务，使用指令 `discard`** 。放弃事务后，事务队列中的所有命令都不会被执行。

---

对于类似 Java 中的编译型异常，也就是**命令的代码有问题的情况下，事务队列中的所有命令都不会被执行**！

比如说，在开启事务后，如果输入到事务队列中的命令里，本来想要输入 `getset [key] [value]` 命令，结果少输入了 1 个 [value] 参数，这样的话，这个事务中所有的命令都会失效：

```bash
127.0.0.1:6379> multi
OK
127.0.0.1:6379> getset k1
(error) ERR wrong number of arguments for 'getset' command
127.0.0.1:6379> set k4 v4
QUEUED
127.0.0.1:6379> exec
(error) EXECABORT Transaction discarded because of previous errors.
```

如果是类似 Java 的运行时异常，也就是**语法上没有问题，但是实际执行的时候出错的情况下，错误的命令不执行，其余没有问题的命令会正常执行**！

比如说，在 `incr [key]` 命令中的 key 应该是一个数字，如果我们让一个 String 进行自增操作，它肯定就会出错。但是在 Redis 事务中，除了 `incr` 指令的那条命令，其余的命令都会照常执行：

```bash
127.0.0.1:6379> multi
OK
127.0.0.1:6379> set k1 v1
QUEUED
127.0.0.1:6379> incr k1
QUEUED
127.0.0.1:6379> set k2 v2
QUEUED
127.0.0.1:6379> get k2
QUEUED
127.0.0.1:6379> exec
1) OK
2) (error) ERR value is not an integer or out of range
3) OK
4) "v2"
127.0.0.1:6379> get k1
"v1"
```

可以这样理解 Redis 的事务：就像我们购买的电子产品，如果开箱后就发现了问题，那么它就是有问题，要执行退货。如果是在使用了一段时间后出现了问题，那么就是某个使用环节的问题，电子产品无法退货。

总之记住：Redis 的事务不保证原子性！

## Redis 监控与乐观锁

悲观锁：

- 认为问题一定会出现，所以无论做什么都加锁，做完后才解锁
- 严重影响性能
- Java 的 `synchronized` 默认就是悲观锁

乐观锁：

- 认为什么时候都不会出现问题，所以不会上锁
- 在更新数据的时候，会去比较判断是否有人改动过数据
- MySQL 的 `version` 字段就是乐观锁。修改数据前获取 version，修改数据完成后再比较 version，然后再选择是否更新

Redis 可以使用 `watch [key]...` 来「监控」这些 key。

比如说，账户内有 1000 元，然后有个专门记账的 key，记录花费了多少钱。我们假设花费了 200 元，那么账户余额就要扣除 200 元（剩余 800），而记账的 key 就要记录增加了 200 元的支出：

```bash
127.0.0.1:6379> set balance 1000
OK
127.0.0.1:6379> set spend 0
OK
127.0.0.1:6379> watch balance
OK
127.0.0.1:6379> multi
OK
127.0.0.1:6379> decrby balance 200
QUEUED
127.0.0.1:6379> incrby spend 200
QUEUED
127.0.0.1:6379> exec
1) (integer) 800
2) (integer) 200
```

这里模拟多线程的情况，现在<u>线程 1 </u>首先*开启监控*，然后在*开启事务*的情况下，进行金额操作，但是*没有 `exec`* ：

```bash
127.0.0.1:6379> watch balance
OK
127.0.0.1:6379> multi
OK
127.0.0.1:6379> decrby balance 10
QUEUED
127.0.0.1:6379> incrby spend 10
QUEUED
```

此时，<u>线程 2 </u>修改了金额，将余额设置为 1000:

```bash
127.0.0.1:6379> get balance
"800"
127.0.0.1:6379> set balance 1000
OK
```

在 <u>线程 2 </u> 进行了操作后，<u>线程 1 </u> 再使用 `exec` 提交事务的话，这个事务就会失败。且金额变为了 <u>线程 2 </u> 修改的 1000：

```bash
127.0.0.1:6379> exec
(nil)
127.0.0.1:6379> get balance
"1000"
127.0.0.1:6379> get spend
"200"
```

也就是说，**在使用乐观锁监控（watch）key 的时候，如果 key 的 value 在事务提交前发生了变化，那么整个事务就无法提交。**

如果加了乐观锁后，事务提交失败，就要先使用 **`unwatch` 指令解除监控**，再重复进行没完成的事务操作：

```bash
127.0.0.1:6379> unwatch
OK
127.0.0.1:6379> watch balance
OK
127.0.0.1:6379> multi
OK
127.0.0.1:6379> decrby balance 10
QUEUED
127.0.0.1:6379> incrby spend 10
QUEUED
127.0.0.1:6379> exec
```

# Redis.conf 配置

在开发和测试时，可以在 redis-cli 内通过：

- `config get [配置名称]` 获取相应的配置信息
- `config set [配置名称] [配置值]` 修改配置信息

生成环境要去修改配置文件 `vim /usr/local/etc/redis.conf`

修改了配置文件后，要重启 redis-server 才会生效（在 redis-cli 输入 shutdown 可以关闭 redis-server）

INCLUDES 分区：可以导入配置文件

NETWORK 分区：

- 使用 `bind 127.0.0.1` 绑定 IP，可以实现远程访问
- 默认使用保护模式 `protected-mode yes`
    - 在 protected mode 下，只有绑定的 ip 或通过密码才能访问
    - 如果设置成 `protected-mode no` ，外部网络就可以直接连接这个 redis
- 端口号设置在 `port 6379`

GENERAL：

- `daemonize no` ：
    - 默认关闭 redis-server 就退出进程，也就是无法后台运行
    - 可以设置 `daemonize yes` 来让它能在关闭后依旧可以运行
- `pidfile /var/run/redis_6379.pid`：
    - 如果 `daemonize yes`，那么就使用这个指令来指定一个 pid 文件
- 日志相关：
    - `loglevel notice` 用来设置日志级别，采用默认的就好了
    - 使用 `logfile ""` 可以设置输出日志的名称，默认空字符串就是标准输出
- `databases 16` ：设置数据库数量为 16 个

---

SNAPSHOTTING（快照）：

这个区域的设置用于持久化，规定在一定时间、一定操作次数后，会持久化到 .rdb / .aof 文件中。

`save [在多少秒内] [进行了多少次操作后就持久化]` ，默认如下：

```
save 900 1
save 300 10
save 60 10000
```

默认压缩 RDB 文件 `rdbcompression yes` ，如果设置为 no，可以节省 CPU 资源。

`dir /usr/local/var/db/redis/` 用于指定 RDB 文件被保存在哪个「目录」（不是文件名，是 directory/目录名）。

---

REPLICATION（复制）：

配置该 Slave 服务器的主机的信息 `replicaof <Master IP> <Master Port>`

如果 Master 有密码，使用 `masterauth [密码]`

---

SECURITY：

- 使用配置文件内的 `requirepass [密码]` 设置密码
- 一般在命令行工具内设置密码，首先进入 redis-cli ：
    - 查看密码：默认没有密码的情况下，可以使用 `config get requirepass` 
    - 设置密码：输入 `config set requirepass [密码]`
    - 输入密码登陆 Redis：`auth [密码]`

CLIENTS：

- `maxclients [可以连接的客户端最大数量]`

MEMORY MANAGEMENT：

- `maxmemory [最大容量（单位 bytes）]`
- `maxmemory-policy noeviction`：
    - 内存满了/到达上限后的处理策略
    - 默认的 noeviction 表示永不过期，返回错误

---

APPEND ONLY MODE（AOF 持久化）：

`appendonly no`：默认不开启 AOF 模式。因为大部分情况下 RDB 完全够用了

`appendfilename "appendonly.aof"`：设置持久化文件的名字

同步机制：

- `appendfsync everysec`：默认每一秒都同步，速度比较快，但是可能会丢失这 1 秒的数据
- `appendfsync always`：每次修改都会同步，但是消耗性能
- `appendfsync no`：这个时候让操作系统来同步数据，速度最快，一般不用

# Redis 持久化

Redis 是内存数据库，如果没有持久化，那么数据就会断电即失，也就是服务器进程退出后，Redis 数据库中的数据也会丢失。

## RDB / Redis DataBase

在指定的时间间隔内，将内存中的数据写入磁盘（Snapshot/快照）。在断电后，可以将快照文件读到内存中，以此来恢复数据。

> Redis 会单独创建（fork）一个子进程来进行持久化。先将数据写入临时文件，等持久化过程结束了，再用临时文件替换以前产生过的持久化文件。这样就能保证主进程不进行 IO 操作，性能极高

基本流程：

1. 父进程/子进程会 fork 一个子进程来处理持久化，父进程继续处理 client 请求
2. 子进程将内容写入临时的 RDB 文件（Snapshot/快照）
3. 子进程将临时的 RDB 文件替换为正式的 RDB 文件（替换旧的 RDB 文件）

优点：

1. 适合大规模的数据恢复
2. 适合对数据恢复的完整性不敏感的情况

缺点：

1. 需要一定的时间间隔进行自动保存的操作
2. 如果 redis 意外宕机，最后一次持久化的数据可能会丢失（因为持久化会先产生临时文件，然后再替换之前的持久化文件）
3. fork 进程的时候，会占用一定的内存空间（以空间换时间）
4. ~~save 命令会消耗主服务器的资源~~

RDB 保存的文件默认名为「dump.rdb」

可以通过在 redis-cli 内输入 `config get dir` 来获取目录，通过 `config set dir [路径]` 来设置目录（此时不需要重启 redis 就能生效）。

---

触发机制：

- 使用 `save` 命令，可以不需要满足条件，直接触发保存 RDB 文件。此时的 RDB 文件不是由子进程生成的，而是由主进程的服务器直接生成
- 使用 `flushall` 命令会生成 RDB 文件。也就是说，清空数据库的命令其实还是会帮助我们保存一份持久化数据（而 `flushdb` 命令不会生成 RDB 文件）
- 退出 redis 的时候，其实就是默认触发了 `save` 命令，所以也会保存 RDB 文件

恢复数据：

- 使用 `config get dir` 获取 dump.rdb 的存放位置（AOF 文件也在这里存放）
- 把保存了「需要被恢复数据的 dump.rdb」文件放入该位置即可自动恢复

## AOF / Append Only File

以日志的形式，将所有命令（除了「读取的操作」）都记录下来，恢复的时候再重新把所有命令执行一遍。

注意：AOF 只会追加文件，不能改写文件

优点：

1. 每一次修改都同步，保证文件的完整性
2. 如果设置为从不同步，效率最高（好像是废话）

缺点：

1. 默认每秒同步一次，所以可能会丢失这 1 秒内的数据
2. AOF 的数据文件远远大于 RDB，修复速度也很慢（因为要重新执行命令）
3. AOF 运行效率低

默认文件名为：「appendonly.aof」

默认是不开启 AOF 的，如果要开启，可以去配置文件内修改，然后重启。也可以在 redis-cli 内：`config set appendonly yes` 。

如果 AOF 文件有错误（比如在文件中手动添加了内容）：

- Redis 就会无法启动
- 此时要修复文件，去「/usr/local/Cellar/redis/5.0.5/bin」内
- 使用命令行指令：`redis-check-aof [--fix] <file.aof>` 修复 AOF 文件
- 默认是输入 `redis-check-aof --fix appendonly.aof` ，然后再输入 `y` （表示 yes）进行修复

# Redis 发布订阅

发布订阅模式：

- 出版-購読型モデル（しゅっぱん-こうどくがたモデル、英: Publish/subscribe）
- Pub/Sub 是一种消息通信模式
- 发送者（Pub）发送消息（message）到频道（Channel/队列）中，订阅者（Sub）从频道（Channel/队列）中接受消息

Redis 发布订阅模式的使用场景：

1. 实时消息系统
2. 实时聊天室（将消息回复给所有人）
3. 订阅、关注系统

复杂的场景使用消息中间件。

注意，要先订阅，才会自动创建频道，之后才能发布。

首先，Sub 订阅一个频道（相当于订阅并创建频道），这里的频道名称为 myChannel：

```bash
127.0.0.1:6379> subscribe myChannel
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "myChannel"
3) (integer) 1
```

然后，Pub 在频道内发送消息：

```bash
127.0.0.1:6379> publish myChannel "this is the first message"
(integer) 1
127.0.0.1:6379> publish myChannel "this is the second message"
(integer) 1
```

此时，在 Sub 端，就会自动接收到消息：

```bash
127.0.0.1:6379> subscribe myChannel
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "myChannel"
3) (integer) 1
1) "message"
2) "myChannel"
3) "this is the first message"
1) "message"
2) "myChannel"
3) "this is the second message"
```

# Redis 主从复制

> 主从复制：将一台 Redis 服务器的数据，复制到其他的 Redis 服务器。前者是主节点/Master，后者是从节点/Slave

**数据的复制是单向的，只能由主节点到从节点**，一般一个 Master 负责处理「写请求」，多个 Slave 负责「读请求」。

每个 Redis 服务器默认都是主节点，且一个主节点可以有多个从节点。不过，一个从节点只能有一个主节点。（One Master, Several Slave）

复制/replication 分类：

- 全量复制：Master 将整个数据库文件发送给 Slave，Slave 将其存盘并加载到内存中
- 增量复制：Master 将新发出修改指令，依次传给各个 Slave，完成同步

复制的原理：

1. Slave 连接到 Master 后会发送一个 sync 同步命令
2. Master 接到命令后，会开始执行 BGSAVE 命令生成 RDB 文件，并使用缓冲区记录此后执行的所有写命令
3. Master 在后台进程执行 BGSAVE 完毕后，<u>Master 会发送整个数据文件到 Slave，完成一次「完全同步/全量复制」</u>

主从复制的作用：

1. 实现了数据的热备份，是持久化之外的一种数据冗余方式
2. 如果主节点出现故障，可以由从节点提供服务，实现快速的故障恢复，实际上是一种服务的冗余
3. 分担服务器负载，实现负载均衡，提高 Redis 服务器的并发量
4. 主从复制是哨兵和集群的基础，也就是说，主从复制是高可用的基础

单台 Redis 服务器最大使用内存不应该超过 20G，如果超过了，就使用集群。

---

主从复制的配置（一主二从）

Redis 服务器默认是 Master，所以只需配置 Slave，让 Slave 认准 Master 即可。

如果要查看当前 Redis 主从复制库的相关信息，使用 `info replication` 命令

这里演示一下在一台服务器内，配置多个 Redis 服务器，配置多台服务器也参照这个写法：

1. 首先找到 `redis.conf` 配置文件，复制多几分，文件名为「redis[端口号].conf」（比如说，redis6379.conf 代表这是端口号为 6379 的 Redis 服务器的配置文件）
2. 然后，修改每一份「redis[端口号].conf」配置文件的信息：
    1. `port [该服务器的端口号]`
    2. `daemonize yes` ：打开后台运行（其实不打开也没问题）
    3. `pidfile /var/run/redis_[该服务器的端口号].pid` ：修改 pid 文件名称
    4. `logfile "[端口号].log"` ：修改日志文件名称
    5. `dbfilename dump[端口号].rdb`：修改 RDB 文件名称
    6. 如果是配置 Slave 服务器的信息，就要设置它的 Master：
        1. `replicaof [Master 的 ip] [Master 的 port]` 
        2. 如果 Master 服务器有密码：`masterauth [密码]`

修改配置文件完毕后，按照配置的端口，打开配置好的 Redis 服务器（比如 `redis-server --port 6380`）和 Redis 客户端（`redis-cli -p 6380`）。

如果要查看打开了哪些 Redis 服务器，输入命令行 `ps -ef | grep redis`

---

使用一主二从（这里采用命令行方式，生产环境下应该在配置文件中设置）：

假设，6379 端口的为 Master，6380 和 6381 端口的为 Slave

进入 6380 端口的 redis-cli，使用指令 `slaveof [host] [port]` 来认定 6370 端口的为 Master：

```bash
127.0.0.1:6380> slaveof 127.0.0.1 6379
OK
```

6381 端口也采用相同操作。此时，使用 `info replication` 指令，从机可以发现 role 已经变成了 slave，主机可以看到 connected_slave 变成了 2。

此时，在 Master 中写入任何内容，Slave 都能读取出来，也就是数据共通了。但是要注意，<u>Slave 不能写入数据</u>。

在没有配置哨兵的情况下，Master 如果断开了，Slave 也依旧是 Slave，不过数据会一直保存着。等 Master 重新开启后，Master 和 Slave 依旧可以无缝连接地照常工作，旧的数据依旧保留，新的数据依旧能够共通。

如果是 Slave 断开了：

- 如果是使用命令行配置的主从信息，那么 Slave 服务器在断开后再重新启动：
    - 原先的 Slave 自己就变成了 Master，无法和原先的 Master 交互
    - 只有重新设置 `slaveof [host] [port]` 才能变回原样
    - 注意：只要设置了主从信息，变成了 Slave，那么数据就是和 Master 互通的，能读取到 Master 的历史数据（全量复制/完全同步）
- 如果使用的是修改配置文件的方式设置主从信息，那么 Slave 断开后再次重连就没有问题

---

既是 Slave 也是 Master

我们还可以将 Slave 设置为另一台服务器的 Master。

比如：

1. 端口 6379 是 Master，6380 是 6379 的 Slave
2. 我们设置 6381 为 6380 的 Slave
3. 此时，6380 既是 6379 的 Slave，也是 6381 的 Master
4. 在 6379 还存在的时候，6380 就还保持着 Slave
5. 如果 6379 断开了，6380 就能够使用 `slaveof no one` 指令切断与 6379 的主从关系，成为新的 Master

就算之前的 Master 再次回归，也没办法回到之前的关系了，只能重新配对。

# Redis Sentinel 哨兵模式

## Redis Sentinel 基础概念

能够在后台监控主机是否故障，如果故障了就根据投票数 **自动将「从库」切换到「主库」**。也就是 Master 如果没了，Slave 会选举新的 Master。

哨兵的监控原理：

- 哨兵是一个单独的进程，会独立运行
- 哨兵通过发送命令，等待 Redis 服务器响应，从而监控多个 Redis 实例

一般采用多哨兵模式：

- 使用多个哨兵进程监控 Redis 服务器
- 各个哨兵之间还会相互监控

哨兵模式运作过程等相关知识，可以[点击这里查看](https://www.cnblogs.com/mzhaox/p/11218096.html) 或 [使用印象笔记打开文章：Redis-哨兵模式和高可用集群解析](https://app.yinxiang.com/shard/s72/nl/16998849/f6b959b4-b09a-4f31-8054-a543aa1213c4/)，关键的地方在于：

1. 主观下线和客观下线，以及以 quorum 为基础的判断机制
2. 故障转移/failover，以及发布订阅模式（Pub/Sub）的应用

优点：

1. 哨兵集群基于主从复制模式，所以继承了其优点
2. 主从服务器可以切换，故障可以转移，提高了系统的可用性
3. 哨兵模式是主从模式的升级版本，完成了手动到自动的升级

缺点：

1. 集群容量一旦到达上限，在线扩容就很麻烦
2. 哨兵模式的配置很多，非常麻烦。如果要修改的话，已经写死的文件修改起来工程量很大

哨兵模式下，如果 Master 进程掉了后，又重新连上了，原先的 Master 就会自动变成 Slave

## Redis Sentinel 快速入门

注意：这里省略了 Slave 和 Master 服务器的配置，这方面的配置在「Redis 主从复制」那个标题下有详细说明。

找到配置文件`redis-sentinel.conf` （Mac 在 /usr/local/etc/ 下，也就是和 redis.conf 在同一个地方），这里有配置的方法。

要使用的话，去在相应的 Redis 实例目录下，创建 `sentinel.conf` 配置文件，然后先加入下面这段配置：

```bash
sentinel monitor <master-name> <ip> <redis-port> <quorum>
port [端口号]
```

其中：

- <master-name>：Master 名称
- <ip>：需要监控的服务器的 IP
- <redis-port>：需要监控的服务器的 port
- <quorum>：当有多少个 sentinel 认为 master 失效时，master 才真的失效（最低通过票数）

之后启动哨兵模式即可。如果是在 Linux 上，就进入 redis 的 bin 文件目录， 在 bin 文件目录下，有 `redis-sentinel` 启动文件（`redis-server` 启动文件也在同一级目录中）和 `kconfig` 目录，该目录包含 `redis.conf` 和 `sentinel.conf` 等 Redis 配置文件。

使用命令 `redis-sentinel kconfig/sentinel.conf` ，表示以 kconfig 目录下的 sentinel.conf 配置文件为基础，启动 redis-sentinel 程序。（记得使用哨兵前，先用 `redis-server kconfig/redis.conf` 来启动 redis-server 服务器）

如果要查看具体配置的话，可以[点击这里](https://paste.ubuntu.com/p/cPtCMSnDkD/)或[使用印象笔记打开：Redis哨兵模式的全部配置](https://app.yinxiang.com/shard/s72/nl/16998849/489716a9-0b2d-4dd5-b665-62394a133198/)

# Redis Cluster 集群模式

## 基础概念

在早期，如果 Redis 要弄多个节点，每个节点存储一部分数据，需要中间件来实现。Redis 3.0 后有原生支持的集群模式 Redis Cluster。

Redis 是存在内存中的，如果容量不够，怎么扩容？

* Redis Cluster 可以缓解内存存储压力（主从复制只能解决读写压力）

在并发「写操作」的时候，Redis 怎么分摊压力？

* 使用 Redis Cluster 集群，来让「写操作（比如 set）」分摊给不同的 Redis 服务器来进行


Cluster：

* クラスターは「房」「集団」「群れ」の意味。ネットワークに接続した複数のコンピューターを連携して1つのコンピューターシステムに統合し、処理や運用を効率化するシステムのこと

Redis Cluster：

* Redis 集群是一个可以在多个 Redis 节点之间进行数据共享的设施（installation），实现了<u>对 Redis 的水平扩容</u>
* Redis 集群通过分区（partition）来提供一定程度的可用性（availability）： 即使集群中有一部分节点失效或者无法进行通讯，集群也可以继续处理命令请求
* Redis 集群不支持那些需要同时处理多个键的 Redis 命令， 因为执行这些命令需要在多个 Redis 节点之间移动数据， 并且在高负载的情况下， 这些命令将降低 Redis 集群的性能， 并导致不可预测的错误

Redis Cluster（日本語）：

* Redisインスタンスをクラスタリングすることができる機能
* クラスター全体であるデータがどのノード(後述)に保存されるかを把握している
    * ノード間でリダイレクト（Redirect）することによって、どのノードから接続しても指定するデータにたどり着ける
* マルチマスター構成を採用していて、データは複数のRedisサーバに自動的に分散される(シャーディング/Sharding)
* マスターがタヒんでも自動的にスレーブがマスターに昇格する(自動フェイルオーバー/Failover)（「タヒんでる」的意思是「死んでる」，因为长得像）

> Redis Cluster はマルチマスター構成をとり、データは複数のRedisサーバに自動的に分散されます(シャーディング/Sharding)。これにより書き込み負荷を分散させることができます。
>
> Redis Cluster は一部のノードが故障しても動作を継続し、ある程度の可用性を提供しますが、多数のノードが利用不可となった場合は動作を停止します。
> 
> 各マスターノードにはスレーブを持たせ、可用性を向上することができます。クラスタの最小構成は3マスターノードとなります。

Redis Cluster 的优点/特点：

* 数据自动分片，会分成 16384 个 Hash Slot，数据能够自动切分（split）到多个节点
* 在部分节点失效时有一定可用性：当集群中的一部分节点失效或者无法进行通讯时，仍然可以继续处理命令请求的能力
* 解决了写操作无法负载均衡，以及存储能力受到单机限制的问题，实现了较为完善高可用

和哨兵模式的对比：

* Redis Cluster 集群不需要哨兵也能完成节点的移除和故障转移
* Redis Cluster 将每个节点设置成集群模式，这种集群没有中心节点，可水平扩展（最多可以线性扩展到 1000 个节点）
* Cluster 集群模式的性能和高可用性均优于之前版本的哨兵模式，且集群配置非常简单

---

相关概念：

Sharding/数据分片：

* Redis 集群使用数据分片（sharding）而非一致性哈希（consistency hashing）来实现
* シャーディングとはDataBaseの負荷分散方法の1種です。水平分割とも呼ばれます。同じテーブルを複数のDataBaseに用意し、1つのテーブルに保存していたレコードを分散する事で各DataBase内に保持されるレコードの量をへらす負荷分散の方法です

シャード/Shard：

* ノードをまとめているグループ
* 1つのシャード内に複数のノードが存在することで、シャード内のノード間でデータが同期(レプリケーション/Replication)される
* 相当于小集群的概念，也就是由 1 个 Master 和多个 Slave 组成的小组

ノード/Node：

* redisインスタンスそのもの（Redis 服务器实例在集群中叫做 node）

マスターノード：

* 2種類のノードのうち、読み込み専用ノード
* マスターノードとデータを共有している
* マスターが死んだら自動でフェイルオーバーする

スロット/slot：

* スロットは0番-16383番まであり、redis clusterで値を格納するとき、格納するデータを0-16383の番号を返すアルゴリズムにかけます
* その返ってきた番号のスロットがあるノードにデータが格納されます。
このようにしてredis clusterは水平分割(シャーディング)をしているのです
* クラスターを組む時、特に指定しなければ*16384/マスターノードの数個*のスロットが各マスターノードに割り振られます

格納する値がかけられるアルゴリズム：

* HASH SLOT = CRC16(key) mod 16384

主从复制模型：

* 为了使得集群在一部分节点下线或者无法与集群的大多数（majority）节点进行通讯的情况下，仍然可以正常运作， Redis 集群对节点使用了主从复制功能
* 集群中的每个节点都有 1 个至 N 个复制品（replica）， 其中一个复制品为主节点（master）， 而其余的 N-1 个复制品为从节点（slave）

参考资料：

https://qiita.com/keitatata/items/44678ad472e61a4606c5

https://www.sraoss.co.jp/tech-blog/redis/redis-cluster/

## Redis Cluster 进阶

Redis Cluster 集群不保证数据强一致性，也就是说，在特定的条件下可能会丢失写操作，有两种会导致丢失数据的情况：

* 情况1：主节点同步数据到从节点之前宕机，从节点升级为主节点，导致数据丢失（因为集群用了「异步复制」）：
    * 假设 Master 收到 redis-cli 的 set key val 命令，在完成 set 操作后，Master 会告诉 redis-cli 操作成功，然后再去异步通知 Slave 进行 set 命令
    * 这种 Master 将写操作，复制给 Slave 的行为，被称为「异步复制」
    * 如果在 Master 告知 redis-cli 命令状态之后，在「异步复制」还没有完成（也就是还没有通知 Slave）之前，Master 宕机了
    * 那么在一段时间之后，会由集群的选举机制选举当前的 Slave 为新的 Master，然而当前的 Slave 没有这条 set key val 的数据
    * 就算到时候当前的 Master 恢复了，也只会以 Slave 的身份同步获得新 Master 的数据，之前 set 操作得来的 key 数据还是会丢失
* 解决情况1：
    * 使用 wait 命令，将客户端阻塞，异步变同步
    * `wait 1`：等 Master 所有写操作成功通知任意 1 个 Slave 之后，再返回给 redis-cli 成功的提示（1 代表需要通知 1 个，如果是 2 就要通知 2 个 slave）
    * `wait 2 1000`：还可以加上超时时间（毫秒），意思是通知了 2 个 slave，并返回给 redis-cli 成功提示的时长，不能超过 1000 毫秒
    * 假设超过了时长，并且没有通知 2 个，而是只通知了 1 个 slave，那么它会返回给客户端数值 1，代表只成功通知了 1 个
    * 注意，我们必须在性能和一致性之间做出权衡，如果使用了 wait 命令，会影响 Master 处理请求的速度
* 情况2：脑裂，也就是集群出现了网络分区，并且一个客户端与至少包括一个主节点在内的少数实例被孤立：
    * 假设有集群[A,B,C,a,b,c]以及客户端，其中大写字母的为 Master，小写字母的为 Slave，如果出现了网络问题（网络分区），集群脑裂为 2 个子集群[A,B,a,b,c]和[C]，且客户端此时只能与 C 连接
    * 在 [A,B,a,b,c] 的子集群中，监测到 C 宕机了，就会选举 c 为新的 Master，等网络恢复后，C 变成了 c 的 slave
    * 如果在网络分裂时期，只能和 C 连接的客户端，刚好对 C 节点有数据写入 key 的操作，这就会导致网络恢复之后，该 key 数据丢失
    * 注意，在网络分裂期间，客户端向能够连接的 Master 发送写命令的时间是有限制的，该时间限制为 node timeout（也就是说，如果该 Master 在一定时间内，还是无法回归集群（感知大多数集群内的 Master），那么该 Master 将变为 error 状态，并无法进行写操作
* 解决情况2：
    * 设定合适的节点超时时间（node timeout）
    * node timeout 会在 2 个情景下发挥作用：
        1. Master 宕机超过 node timeout 后，就会选举 Slave 为新的 Master
        2. 某个 Master 无法感知集群内的其他 Master 后，如果超过了 node timeout 还没办法回归集群，就转为 error 状态，并停止写入功能

Redis Cluster 采用的方案：

* 采用主从加选举方案，保证高可用
* 采用 Sharding（类似一致性哈希，但不是）的方式，保证数据容量可以横向拓展，只移动部分元素就可以实现动态添加删除节点，也就是使用了哈希槽/Hash Slot

![](https://user-gold-cdn.xitu.io/2017/7/10/ab9498763e328b93d17bdcd93f20f4eb?im)

如果管理某一个 slot 的 Master 和其 Slave 都挂掉了，这时如果需要执行 set/get 操作的 key，刚好要转移到该 slot 下，就会出现「The cluster is down」的提示，表示无法添加/获取该 key：

* 这个时候要在 redis.conf 配置文件中，设置 `cluster-require-full-coverage no`
* 这个配置的意思是「只有 16384 个 slot 都处于正常状态的时候，才对外提供服务」
* 设置为 no 的话，可以保证部分 slot 无法运转后，这个 Cluster 集群还能完成数据操作

Redis cluster 每个节点都需要开启两个 TCP 端口。Clusterが使用するポートは通常のポートに加え、クラスタの通信用に+10000したポート番号を使用します(6379の場合は16739)：

* redis服务端和客户端通信的端口，比如配置端口6379，响应的redis会使用 6379+10000=16379端口，作为集群总线，用于集群间数据交互
* 10000 这个数字是固定的。所以搭建redis集群的时候要确保防火墙放开了客户端端口和集群总线端口。否则客户端功能或者集群功能将无法正常使用。

不在一个 slot 下的 key，无法完成 mget 和 mset 这样的批量操作，这里可以使用 Hash Tag，也就是通过 `{tag名称}` 来分组，从而使相同 tag 的 key 放到同一个 slot 中。（key 可以写成 {vip}user、user{vip} 和 user{vip}name 等等格式）

Hash Tag：

> hash Tag是redis集群比较好用的特性。当然这个功能我们也可以再客户端这边自己去实现。简单来讲hash tag就是可以标记key中用hash算法生成hash槽的部分。从而控制特定的key被分配到特定的hash槽中，从而可使用redis的批量命令（mget,mset等），这个功能的问题也比较明显，就是有可能会导致数据分配不均。
>
> 使用方式就是把key中需要用来做hash的部分用{}括起来，比如 {englishClass}_zhangsan, {englishClass}_lisi。 这两个key会被分配到一个hash槽中，从而可以用mget批量获取一个班级的所有学生。

如果想要计算 Hash Slot，可以在 redis-cli 中使用指令：

* `cluster keyslot [key名称]`
* 它会返回 hash slot 的值

如果想查询某个槽位/slot 目前的 key 的数量，使用指令：

* `cluster countkeysinslot [slot名称（数值）]`
* 命令的意思是 count keys in slot

知道了某个 slot 下有多少个 keys 后，我们可以获取该 slot 下所有的 keys：

* `cluster getkeysinslot [slot名称] [数量]`
* 获取 [数量] 个 [slot名称] 下的 keys

manual failover：

> 因为redis集群不保证强一致性，存在数据丢失风险，所以如果想要主动下掉某个主节点的话，其实是有数据丢失风险的。这个时候，manual failover 就可以起到作用。
>
> 如果某个主节点出现问题，想要重启，或者迁移之类的操作的话，可以使用redis 的CLUSTER FAILOVER 命令。从想要下线的主节点的从节点发起命令，主节点和从节点会调换身份。期间会阻塞客户端请求一段时间，保证数据一致后，开始转换，从而保证数据一致性。

转换到 Cluster 模式：

* 该功能支持从单主节点 redis 数据转换成 redis cluster 模式
* 假设有 5 个主节点，主要复制数据到 5 个主节点就行，从节点可以后期添加（从节点不会影响数据分布）：
    1. 停掉客户端
    2. 使用 BGREWRITEAOF 命令，保证数据都同步到 AOF 文件
    3. 把 AOF 文件保存 5 份
    4. 创建 5 个主节点，确定是 AOF 的持久化方式，后期再添加从节点
    5. 停掉所有节点，配置他们的 AOF 文件为刚才创建的 AOF 文件，每个节点一份
    6. 重启所有节点，让其加载对应的配置文件
    7. 用 redis-cli --cluster fix 命令让数据在各节点之前迁移
    8. 用 redis-cli --cluster check 命令检查集群状态
    9. 重启客户端

## Redis Cluster 的使用和配置

在 set 一个 key 的时候，首先会对 key 进行 crc16 哈希算法，得出一个 hash 值，然后将 hash 值 mod（%）16384 得出槽位/slot：

* set hello world 的 key 是 hello，然后使用 crc16 算法：crc16(hello) = 50018
* 计算 slot：50018 % 16384 = 866
* <u>Redis 规定一共有 16384 个 slots</u>

这里假设有 3 个「一主二从」的小集群。那么第 1 个小集群（其实是主节点/Master），会负责 0 到 5461 的 slot，第 2 个小集群负责 5462 到 10922 的 slot，第 3 个小集群负责 10923 到 16384 的 slot（注意，这里是为了方便，才将总数 16384 分为 3 个区间，实际上可能第 1 个小集群负责 0 到 20，100 到 120……）。

我们计算出来 hello 的 key 为 866，是由第 1 个小集群负责的，所以这个 set 命令会发送到第 1 个小集群

注意：**只有 Master/主节点才会对应 slot/槽位**，上面说的小集群不是严谨的概念，只是为了方便理解而已。

---

以下是配置相关，每个集群内的 Redis 服务器都要对应 1 个 redis[端口号].conf（或者改名为 nodes-[端口号].conf）

设置的时候，除了一些主从配置（参考之前的笔记），还要找到 redis.conf 的 REDIS CLUSTER 的区域：

* `cluster-enabled yes`
* `cluster-config-file nodes-6379.conf`：记录集群信息，不用手动维护，Redis Cluster 会自动维护（6379 是当前节点，可以修改）
* `cluster-node-timeout 15000`：Cluster 集群节点连接超时时间（超过时间自动主从切换），如果集群规模小，都在同一个网络环境下，可以配置的短些，更快的做故障转移
* `cluster-replica-validity-factor 0`：
    * 如果 Master 和 Slave 都挂掉，而且 Slave 和 Master 失联又超过「这个数值 * timeout」的时间，就再也不会发起选举
    * 如果设置为 0，就是永远都会尝试发起选举，也就是一直尝试从 slave 变为 master
* `cluster-require-full-coverage no`：
    * 默认为 yes，需要设置为 no
    * 当 Master 宕机且 Slave 还没有完成故障转移的期间，整个集群是否不可用
    * 如果设置为 yes，只要故障转移还没有完成，整个集群就无法使用；对于大多数业务无法容忍这种情况，因此要设置为 no
    * 当主节点故障时，只影响它负责槽的相关命令执行，不会影响其他主节点的可用性
    * 只要有结点宕机导致 16384 个槽没全被覆盖，整个集群就全部停止服务，所以一定要改为 no

如果是在单机上批量修改的话，可以使用：

* `sed 's/7000/7001/g' ./redis7000/redis.conf > ./redis7001/redis.conf`
* 把所有「7000」的部分，替换为「7001」，并生成新的文件，源文件是 redis7000 文件夹下的 redis.conf，新生成的文件放在 redis7001 目录下，命名为 redis.conf

---

在 redis-cli 中使用 `cluster nodes` 指令，可以查看当前集群下，相关节点的信息。

如果当前集群下没有节点，就需要自己搭建。在 Redis 5.0 之后，可以使用 redis-cli 直接设置集群节点，在命令行输入 `redis-cli --cluster help` 可以 list all available cluster manager commands，然后根据提示来设置即可：

`redis-cli --cluster create 127.0.0.1:6379 127.0.0.1:6380 127.0.0.1:6381 --cluster-replicas 2`：

* 在命令行使用创建 Cluster 的节点的命令，跟着 Redis 服务器的 IP 和 端口
* ，然后「--cluster-replicas <参数>」用于分配主从：
    * 如果参数为「1」，代表将服务器自动按照 1 个 Master 跟着 1 个 Slave 的比例分配
    * 如果参数为「2」，代表将服务器自动按照 1 个 Master 跟着 2 个 Slave 的比例分配
    * 默认使用命令行内，顺序较前的服务器作为 Master，比如这里就是 6379 端口的服务器作为主服务器
    * 注意，假设有 6 个 Redis 服务器，那么参数只能为「1」，也就是 3 个 Master 和 3 个 Slave
    * ，因为<u>Redis Cluster 需要至少 3 个 Master</u>，如果参数为「2」，那么就变成 2 个 Master 和 4 个 Slave，Master 就少于了 3 个

在输入上面那行命令后，命令行会询问你是否确定，我们手动回复 `yes`，它就开始自动创建集群（slot 之类的会自动分配）。

---

在 Redis Cluster 搭建完成后，我们不能随便 set key 了，而要到该 key 相应的 slot 所属的 Redis 服务器的 redis-cli 内才能 set key。

如果要正常使用，就使用命令行 `/[目录]/redis-cli -h [IP地址] -p [端口号] -c` 的命令（也就是加上 `-c`，c 代表 cluster）进入集群模式的命令行。

此时，如果你 set 的 key 的 slot 不在这台服务器上，它会自动将这个 set 指令，Redirect to slot [正确的槽位] located at [正确的服务器 IP 和端口]。也就是说，就算你进入了 6379 端口的 redis-cli，如果你 set 的 key 应该在 6380 执行，它会自动将 redis-cli 移动到 6380 端口，然后自动执行 set 命令。

---

Redis Cluster 的扩容可以使用上面介绍的 redis-cli --cluster help 后，找到 add-node 相关的指令：

`redis-cli --cluster add-node [新服务器的IP和端口] [已存在集群内的任意服务器的IP和端口]`：

* 用于在 Cluster 中添加新的 Master
* 比如说，新的服务器IP和端口为 127.0.0.1:7000，已经在集群内的任意一台服务器的IP和端口为 127.0.0.1:6379
* 我们要添加这个新的 Redis 服务器到集群中，就可以输入 `redis-cli --cluster add-note 127.0.0.1:7000 127.0.0.1:6379`

`redis-cli --cluster add-node [新服务器IP和端口] [已存的任意服务器IP和端口] --cluster-slave --cluster-master-id [该Slave的Master的ID]`：

* 用于在 Cluster 中添加新的 Slave
* [该Slave的Master的ID] 可以通过 `cluster nodes` 指令来查看

注意，此时在 Cluster 中新添加的 Redis 服务器还没有分配 Slots，所以需要重新分配 slots，我们可以通过之前提到 redis-cli --cluster help 命令，找到 reshard 相关的指令来：

`redis-cli --cluster reshard [服务器IP和端口]`：

* 这里的[服务器IP和端口]，指的是已经在 Cluster 集群中，且已经有了分配好的 slots 的 Redis 服务器（任意一个即可，这里会将该集群内所有的 slots 重新进行分配）
* 现在要做的，是把已经分配好的 slots，再重新分配给新添加的 Redis 服务器，所以命令行还会继续问你：
    1. 要将现有的 Cluster 集群内，已经分配好的 slots，分出多个给新添加的 Redis 服务器？回复的时候填上 1 到 16384 之间的数值
    2. 需要分配给哪个新添加的 Redis 服务器？回复的时候填上该服务器的 ID（注意，只有 Master 才能拥有 slots）
    3. 然后会问你，将现有的哪台服务器的 slots 分配给新的服务器？
        * 回复 `all`：现在已经分配好 slots 的服务器，会平均转让部分 slots 给新的服务器（具体的数量看第 1 个问题）
        * 回复某个 Redis 服务器的 ID：让该服务器转让部分 slots 给新的服务器，然后回复 `done`
        * 如果回复某个 Redis 服务器的 ID 后，还想继续添加，就先别回复 `done`，而是继续输入 ID，这样新输入的服务器也会转让 slots，最后再输入 `done`
    4. 设置完成后，会问你是否确认？回复 yes 即可。

> 注意，slot/槽位被转让了之后，该 slot 下的数据也会被转移到新的服务器上
> 
> 各ノードのhash slotは後から自由に移動することができるため、後からノードを追加したりノードを削除することが可能です。


---

除了扩容，还能「缩容」，也就是减少集群下的 Redis 服务器。

首先要把需要剔除的服务器下，所有的 slots 转移出去：

`redis-cli --cluster reshard [该集群下任意1个服务器的IP和端口] --cluster-from [要转让slots的服务器ID] --cluster-to [要接收slots的服务器的ID] --cluster-slots [转让的slots的数量] ` （可以选择在命令行中，手动回复「yes」，或者直接在命令中添加参数「--cluster-yes」）

然后，在 Redis Cluster 集群中，删除相应的服务器节点：

`redis-cli --cluster del-node [集群中任意1个服务器的IP和端口] [需要删除的服务器的ID]`

注意：

1. 转移 slots 的时候，只转移 Master 即可（因为只有 Master 有 slots）
2. 移除服务器的时候，要先移除 Slave，再移除 Master（如果先移除 Master，会触发故障转移机制，影响不大，但是会有没必要的资源消耗）

> 将一个哈希槽从一个节点移动到另一个节点不会造成节点阻塞，所以无论是添加新节点还是移除已存在节点，又或者改变某个节点包含的哈希槽数量，都不会造成集群下线

# Redis 缓存穿透、击穿和雪崩

> 服务的高可用问题

缓存穿透（查不到）：

- 如果用户想要查询的数据，在 Redis 缓存中找不到，就叫做缓存没有命中（没有命中叫做 miss，命中叫做 hit）
- 此时就会去持久层的数据库（MySQL）中查询该数据，如果在持久层数据库中也找不到这个数据，那么就是查询失败
- 如果有大量的请求（比如，秒杀的场景），缓存都没有命中，就会给持久层造成很大的压力，这就是「缓存穿透」

缓存穿透的解决思路：

1. 思路一：在接受请求的时候，设置一个过滤器
2. 思路二：如果缓存没有命中，就存储该对象为「空」，这样再次查询的时候，就通过缓存来返回「空」

缓存穿透的解决方案：

1. BloomFilter/布隆过滤器：
    - 是一种数据结构，对所有可能查询的参数以 hash 形式存储
    - 在控制层先校验，如果 miss 就直接返回给客户端，只有 hit 了让请求走到缓存和持久层
2. 缓存空对象：
    - 请求在缓存层和存储层都 miss 的情况下，缓存就去 cache null values（将该对象设为 key，value 为 null）
    - 同时会设置该 key 的到期时间

缓存为空对象的问题：

1. 浪费很多空间给 key 去存 null value(s)
2. 即使设置了过期时间啊，还是会存在缓存层和存储层在一定时间内的不一致

---

缓存击穿（量太大，遇上了缓存过期）：

- 如果大量的并发请求集中访问一个点，且在缓存里这个 key 设置了过期时间
- 那么在 key 因为过期而失效，到重新生效的瞬间，持续的大并发就会击穿缓存，直接访问数据库
- 此时，大量请求访问数据库并回写缓存的话，数据库压力会很大

解决思路：

1. 设置热点 key 不过期
2. 使用「分布式锁」：
    - 在某个 key 过期的时候，会有很多个线程访问数据库
    - 加上分布式锁，只让其中一个有权限的线程访问数据库

---

> 缓存雪崩：在某段时间内，缓存集中过期失效，或 Redis 服务器宕机

比如在双十一的时候，把商品数据存在了缓存中，设置到凌晨一点过期，等到了时间这批商品的数据就会集体过期，对这批商品的访问查询，全部都到了数据库中。持久层的数据库会产生周期性的压力波峰，也会出现挂掉的情况。

1. 在缓存雪崩中，key 集中过期还算可以接受，数据库会产生周期性的压力而已
2. 但是如果是缓存服务器某个节点宕机或断网而形成的「自然雪崩」就很致命了，很可能瞬间压垮数据库

解决方案：

- Redis 高可用
    - 增加多几台 Redis 服务器
    - 也就是搭建集群，异地多活
- 限流降级
    - 通过加锁或队列来控制操作数据库的线程数量
    - 比如某个 key 只允许一个线程查询数据和写入缓存
- 数据预热：
    - 在正式部署前，先把可能的数据预先访问一遍，让大量访问数据加载到缓存中
    - 在即将发生大并发访问前手动触发加载缓存 key，设置不同的/随机的过期时间，让失效时间均有分布
- 不同的模块设置不同的时效：
    - 把业务分为不同的模块，也就是将 key 分类，不同的 key 设置不同的缓存超时时间
- 其他思路：
    - 停掉一些服务，保证主要的服务可用
    - 比如双十一的时候，停掉了退款的功能

