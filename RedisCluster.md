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