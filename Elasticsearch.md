[toc]

# Elasticsearch 基础

## Elasticsearch 基础概念

概念：

> Elasticsearch は Elastic 社が開発しているオープンソースの全文検索エンジンです
> Elasticsearch（エラスティックサーチ）はLucene基盤の分散処理マルチテナント対応検索エンジンである
> 
> Elasticsearch是一个基于Lucene的搜索服务器。它提供了一个分布式多用户能力的全文搜索引擎，基于RESTful web接口。Elasticsearch是用Java语言开发的，并作为Apache许可条款下的开放源码发布，是一种流行的企业级搜索引擎
> Elasticsearch 是一个分布式、高扩展、高实时的搜索与数据分析引擎。它能很方便的使大量数据具有搜索、分析和探索的能力。充分利用Elasticsearch的水平伸缩性，能使数据在生产环境变得更有价值

作用：

> 大量のドキュメントから目的の単語を含むドキュメントを高速に抽出することができます：
>
> 全文検索に特化しており、他のソリューションと比較しても圧倒的な全文検索スピードと利便性を誇る

特点：

> Elasticsearchの内部ではApache Luceneが提供する超高速全文検索をフル活用しており、スケーラブル、スキーマレス、マルチテナントを特長とする
>
> Elasticsearch では RESTful インターフェースを使って操作しますが、「Elasticsearch SQL」を使って SQL 文でクエリを記述することもできます。

原理：

> Elasticsearch 的实现原理主要分为以下几个步骤，首先用户将数据提交到 Elasticsearch 数据库中，再通过分词控制器去将对应的语句分词，将其权重和分词结果一并存入数据，当用户搜索数据时候，再根据权重将结果排名，打分，再将返回结果呈现给用户

Elasticsearch is a distributed RESTful search engine built for the cloud. Features include:

* Distributed and Highly Available Search Engine.
    * Each index is fully sharded with a configurable number of shards.
    * Each shard can have one or more replicas.
    * Read / Search operations performed on any of the replica shards.
* Multi Tenant.
    * Support for more than one index.
    * Index level configuration (number of shards, index storage, ...).
* Various set of APIs
    * HTTP RESTful API
    * All APIs perform automatic node operation rerouting.
* Document oriented
    * No need for upfront schema definition.
    * Schema can be defined for customization of the indexing process.
* Reliable, Asynchronous Write Behind for long term persistency.
* (Near) Real Time Search.
* Built on top of Apache Lucene
    * Each shard is a fully functional Lucene index
    * All the power of Lucene easily exposed through simple configuration / plugins.
* Per operation consistency
    * Single document level operations are atomic, consistent, isolated and durable.

参考：[はじめての Elasticsearch](https://qiita.com/nskydiving/items/1c2dc4e0b9c98d164329)

注意：Elasticsearch 属于 NoSQL

## Elasticsearch 的下载安装、基础配置、后台运行

镜像下载地址：

* ElasticSearch: https://mirrors.huaweicloud.com/elasticsearch/?C=N&O=D
* logstash: https://mirrors.huaweicloud.com/logstash/?C=N&O=D
* kibana: https://mirrors.huaweicloud.com/kibana/?C=N&O=D

这里使用 Linux：

使用 `wget -c [链接]` 断点下载，然后使用 `tar -zxvf [压缩文件]` 解压缩 tar.gz 文件。

配置文件在 elasticsearch-7.9.3/config 目录下：

* elasticsearch.yml：es 的配置文件
* jvm.options：JVM 配置
* log4j2.properties：日志配置
* plugins：存放插件
* logs：日志
* modules：功能模块

使用 `free -h` 查看内存大小，然后去 jvm.options 内设置内存，比如最小内存设置为 512m：`-Xms512m`

不能使用 root 用户启动 elasticsearch，所以要创建并使用新用户来启动

* 记得使用 root 用户给普通用户授权 `chown -R 用户名:用户名 /usr/local/elasticsearch`

远程访问，需要在 elasticsearch.yml 设置：

* network.host: 0.0.0.0
* 也可能需要使用这个：cluster.initial_master_nodes: ["node-1", "node-2"]

出现的错误以及解决方法：

* ERROR : [2] bootstrap checks failed
* [1] : initial heap size [536870912] not equal to maximum heap size [1073741824]; this can cause resize pauses
    * ES 认为现在是生产环境（集群），启动的时候会对环境做检查，所以要将状态设置为单节点状态/单个实例状态
    * 在 elasticsearch.yml 中添加：discovery.type: single-node
* max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
    * 注意，上面设置了 discovery.type: single-node 后，应该就不会出现这个错误了
    * 最大虚拟内存太小，解决办法切换到 root 用户修改配置 sysctl.conf
    * vim /etc/sysctl.conf 后添加：vm.max_map_count=262144

可以使用 chrome 的 ElasticSearch Head 插件来作为 ES 的图形界面。

如果在服务器上，有其他软件（比如在服务上安装了 Head 插件）需要访问 ElasticSearch，可能会出现跨域问题，这就要在 elasticsearch.yml 中添加：

```yml
http.cors.enabled: true
http.cors.allow-origin: "*"
```

解决跨域问题，还可以：

```java
@Configuration
public class WebMvcConfig extends WebMvcConfigurerAdapter {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
                .allowedOrigins("*")
                .allowedMethods("POST", "GET", "PUT", "OPTIONS", "DELETE")
                .maxAge(3600)
                .allowCredentials(true);
    }
}
```

参考：[SpringBoot配置Cors解决跨域请求问题](https://www.cnblogs.com/yuansc/p/9076604.html)或用印象笔记打开：[SpringBoot配置Cors解决跨域请求问题 - 袁老板 - 博客园](https://app.yinxiang.com/shard/s72/nl/16998849/b3d4fbe2-70fe-4c86-8ea1-eae1dc8e271d/)

后台运行：`./bin/elasticsearch -d`

## Elastic Stack（ELK）

Elastic Stack とは

> Elastic Stack は Elasticsearch 関連製品の総称です。
> バージョン 2.x 以前は「ELK」と呼ばれていましたが、バージョン 5 からは名称を改め「Elastic Stack」となりました。
> 
> Elasticsearch是与名为Logstash的数据收集和日志解析引擎以及名为Kibana的分析和可视化平台一起开发的。这三个产品被设计成一个集成解决方案，称为“Elastic Stack”（以前称为“ELK stack”）

| 製品名 | 機能 |
| --- | --- |
| Elasticsearch | ドキュメントを保存・検索します |
| Logstash | データソースからデータを取り込み・変換します |
| Kibana | データを可視化します |
| Beats | データソースからデータを取り込みます |
| X-Pack | セキュリティ、モニタリング、ウォッチ、レポート、グラフの機能を拡張します |

Logstash 是 ELK 的中央数据流引擎，从不同目标（文件/数据存储/MQ）收集不同格式的数据，过滤后输出到不同的目的地（文件/MQ/Redis/Elasticsearch/Kafka）

参考：[はじめての Logstash](https://qiita.com/nskydiving/items/0cb598de7ffb5c22424d)

Kibana 可以将 Elasticsearch 的数据通过友好的界面展示出来，提供实时分析的功能

Elastic Stack（ELK）一般被认为是日志分析架构技术栈总称，但它不仅适用于日志分析，还可以支持任何数据的分析和收集。

## Kibana

Kibana 相当于一个 Elasticsearch 的开发工具。

安装的时候，Kibana 和 Elasticsearch 的版本号必须一致。

* 如果系统内存不够，可以 `vim bin/kibana`，然后找到 NODE_OPTIONS=" "，在这个里面添加 --max-old-space-size=700，表示该程序可用内存缩小到 700MB

初次启动，需要在 kibana.yml 中：

* server.port: 5601 （打开端口，其实默认也打开了）
* server.host: （这是打开 kibana 的 IP）
    * 服务器："0.0.0.0" （如果在服务器上使用，并且能让外部访问，需要这样设置）
    * 自己的电脑："localhost"
* elasticsearch.hosts: ["http://47.111.240.178:9200"] （这是 elasticsearch 的地址）

# Elasticsearch 进阶

## Elasticsearch 核心概念

核心：

1. 索引
2. 字段类型（Mapping）
3. 文档 Document
4. 分片（Inverted Index）

关系型数据库和 Elasticsearch 的对比：

| Relational DB | Elasticsearch |
| -- | -- |
| Databases 数据库 | Indices 索引 |
| Tables 表 | ~~Types（弃用）~~ |
| Rows 行 | Documents 文档 |
| Columns 字段 | Fields |

Elasticsearch 在后台会把索引（Indices）分成多个分片（Shards/シャード），每个分片可用在集群中的不同服务器间迁移。

**Elasticsearch 是面向文档的**，索引和搜索数据的最小单位是文档。

## Inverted Index

Inverted Index

**Inverted index 是将单词或记录作为索引，将文档 ID 作为记录，这样便可以方便地通过单词或记录查找到其所在的文档** ，虽然被称为倒排索引，但是翻译成转置索引更合适：

* 正排索引(forward index)：通过 key 找 value
    * 从文档角度看其中的单词，表示每个文档（用文档ID标识）都含有哪些单词，以及每个单词出现了多少次（词频）及其出现位置（相对于文档首部的偏移量）
* 倒排索引(inverted index，或inverted files)：通过 value 找 key
    * 从单词角度看文档，标识每个单词分别在那些文档中出现(文档ID)，以及在各自的文档中每个单词分别出现了多少次（词频）及其出现位置（相对于该文档首部的偏移量）。


> 転置インデックス（てんちインデックス、Inverted index）とは、全文検索を行う対象となる文書群から単語の位置情報を格納するための索引構造をいう。転置索引、転置ファイル、逆引き索引などとも呼ばれる。
> 
> 倒排索引，也常被称为反向索引、置入档案或反向档案，是一种索引方法，被用来存储在全文搜索下某个单词在一个文档或者一组文档中的存储位置的映射。它是文档检索系统中最常用的数据结构。

参考：

* [倒排索引为什么叫倒排索引？](https://www.zhihu.com/question/23202010)
* 维基百科

[示例：](https://www.elastic.co/guide/cn/elasticsearch/guide/current/inverted-index.html)

例如，假设我们有两个文档，每个文档的 *content* 域包含如下内容：

1. The quick brown fox jumped over the lazy dog
2. Quick brown foxes leap over lazy dogs in summer

为了创建倒排索引，我们首先将每个文档的 content 域拆分成单独的 *词*（我们称它为 *词条* 或 *tokens* ），创建一个包含所有不重复词条的排序列表，然后列出每个词条出现在哪个文档。结果如下所示：

```
Term      Doc_1  Doc_2
-------------------------
Quick   |       |  X
The     |   X   |
brown   |   X   |  X
dog     |   X   |
dogs    |       |  X
fox     |   X   |
foxes   |       |  X
in      |       |  X
jumped  |   X   |
lazy    |   X   |  X
leap    |       |  X
over    |   X   |  X
quick   |   X   |
summer  |       |  X
the     |   X   |
```

现在，如果我们想搜索 quick brown ，我们只需要查找包含每个词条的文档：

```
Term      Doc_1  Doc_2
-------------------------
brown   |   X   |  X
quick   |   X   |
------------------------
Total   |   2   |  1
```

两个文档都匹配，但是第一个文档比第二个匹配度更高。如果我们使用仅计算匹配词条数量的简单 *相似性算法* ，那么，我们可以说，对于我们查询的相关性来讲，第一个文档比第二个文档更佳。

但是，我们目前的倒排索引有一些问题：

* Quick 和 quick 以独立的词条出现，然而用户可能认为它们是相同的词。
* fox 和 foxes 非常相似, 就像 dog 和 dogs ；他们有相同的词根。
* jumped 和 leap, 尽管没有相同的词根，但他们的意思很相近。他们是同义词。

使用前面的索引搜索 *+Quick +fox* 不会得到任何匹配文档。（ *+* 前缀表明这个词必须存在。）只有同时出现 *Quick* 和 *fox* 的文档才满足这个查询条件，但是第一个文档包含 *quick fox* ，第二个文档包含 *Quick foxes* 。

如果我们将词条规范为标准模式，那么我们可以找到与用户搜索的词条不完全一致，但具有足够相关性的文档。例如：

* *Quick* 可以小写化为 *quick* 。
* *foxes* 可以 *词干提取* --变为词根的格式-- 为 *fox* 。类似的， *dogs* 可以为提取为 *dog* 。
* *jumped* 和 *leap* 是同义词，可以索引为相同的单词 *jump* 。

现在索引看上去像这样：

```
Term      Doc_1  Doc_2
-------------------------
brown   |   X   |  X
dog     |   X   |  X
fox     |   X   |  X
in      |       |  X
jump    |   X   |  X
lazy    |   X   |  X
over    |   X   |  X
quick   |   X   |  X
summer  |       |  X
the     |   X   |  X
```

这还远远不够。我们搜索 *+Quick +fox* 仍然 会失败，因为在我们的索引中，已经没有 *Quick* 了。但是，如果我们对搜索的字符串使用与 content 域相同的标准化规则，会变成查询 *+quick +fox* ，这样两个文档都会匹配！

***

Inverted Index 相关概念：

* 文档(Document)：一般搜索引擎的处理对象是互联网网页，而文档这个概念要更宽泛些，代表以文本形式存在的存储对象，相比网页来说，涵盖更多种形式，比如Word，PDF，html，XML等不同格式的文件都可以称之为文档。再比如一封邮件，一条短信，一条微博也可以称之为文档。在本书后续内容，很多情况下会使用文档来表征文本信息。
* 文档集合(Document Collection)：由若干文档构成的集合称之为文档集合。比如海量的互联网网页或者说大量的电子邮件都是文档集合的具体例子。
* 文档编号(Document ID)：在搜索引擎内部，会将文档集合内每个文档赋予一个唯一的内部编号，以此编号来作为这个文档的唯一标识，这样方便内部处理，每个文档的内部编号即称之为“文档编号”，后文有时会用DocID来便捷地代表文档编号。
* 单词编号(Word ID)：与文档编号类似，搜索引擎内部以唯一的编号来表征某个单词，单词编号可以作为某个单词的唯一表征。
* 倒排索引(Inverted Index)：倒排索引是实现“单词-文档矩阵”的一种具体存储形式，通过倒排索引，可以根据单词快速获取包含这个单词的文档列表。倒排索引主要由两个部分组成：“单词词典”和“倒排文件”。
* 单词词典(Lexicon)：搜索引擎的通常索引单位是单词，单词词典是由文档集合中出现过的所有单词构成的字符串集合，单词词典内每条索引项记载单词本身的一些信息以及指向“倒排列表”的指针。
* 倒排列表(PostingList)：倒排列表记载了出现过某个单词的所有文档的文档列表及单词在该文档中出现的位置信息，每条记录称为一个倒排项(Posting)。根据倒排列表，即可获知哪些文档包含某个单词。
* 倒排文件(Inverted File)：所有单词的倒排列表往往顺序地存储在磁盘的某个文件里，这个文件即被称之为倒排文件，倒排文件是存储倒排索引的物理文件。

参考：[什么是倒排索引？](https://www.cnblogs.com/zlslch/p/6440114.html)

## IK 中文分词器

在 IK 分词器的 [GitHub](https://github.com/medcl/elasticsearch-analysis-ik)，找到下载地址，然后找到类似 elasticsearch-analysis-ik-7.8.0.zip（注意需要的版本号，以及是 zip 文件）的文件，下载完成后放到 Elasticsearch 的 plugins 文件夹下。

* 日本語/日语/日文/Japanese 使用 kuromoji 分词器
* 7.8.0 版本直接下载地址：https://artifacts.elastic.co/downloads/elasticsearch-plugins/analysis-kuromoji/analysis-kuromoji-7.8.0.zip
* 安装参考：
    * [（Mac）homebrewでElasticsearch 7.8.0とanalysis-kuromojiプラグインのインストール](https://pointsandlines.jp/env-tool/mac/elasticsearch-install)
    * [Japanese (kuromoji) Analysis Pluginedit](https://www.elastic.co/guide/en/elasticsearch/plugins/current/analysis-kuromoji.html)
* 使用参考：
    * [Elasticsearch で kuromoji を使って Kibana でタグクラウドを作る](https://mattintosh.hatenablog.com/entry/20190205/1549371581)
    * [Elasticsearchで簡単に検索できるまでたどり着く方法](https://note.com/ktphy/n/n357e62f3d5cc)

> 分词器的英文是 Analyzer, which is a combination of tokenizer and filters that can be applied to any field for analyzing in elasticsearch：
> * [What is tokenizer, analyzer and filter in Elasticsearch ?](https://arjunjs.wordpress.com/2018/02/06/what-is-tokenizer-analyzer-and-filter-in-elasticsearch/)
> * [ElasticSearch の全文検索での analyzer について](https://blog.chocolapod.net/momokan/entry/115)

使用 IK 插件的时候，直接重启 Elasticsearch 即可，如果要查看 Elasticsearch 的所有插件，可以使用命令： ./bin/elasticsearch-plugin list

测试的时候，使用 kibana 的 Dev Tools，比如

```http
GET _analyze
{
  "analyzer": "ik_smart",
  "text": "正在做一个搜索引擎"
}
```

使用 `ik_smart` 或 `ik_max_word`（注：ik_max_word 比 ik_smart 更精细）将 text 的内容进行分词处理：

```http
{
  "tokens" : [
    {
      "token" : "正在",
      "start_offset" : 0,
      "end_offset" : 2,
      "type" : "CN_WORD",
      "position" : 0
    },
    {
      "token" : "做一个",
      "start_offset" : 2,
      "end_offset" : 5,
      "type" : "CN_WORD",
      "position" : 1
    },
    {
      "token" : "搜索引擎",
      "start_offset" : 5,
      "end_offset" : 9,
      "type" : "CN_WORD",
      "position" : 2
    }
  ]
}
```

如果需要配置自己的词典/字典

* 在 config 中创建一个 dic 结尾的文件（比如hello.dic），然后在里面直接编写就可以了（一行代表一个词）
* 到 config/IKAnalyzer.cfg.xml 里面，将自己的字典 hello.dic 加入配置中：<entry key="ext_dict">hello.dic</entry>

# Elasticsearch 的操作

## 基础 REST 风格操作

**PUT**

<u>创建索引并存放数据</u>

使用 Kibana 的开发工具的话，创建索引并存放的格式是：

PUT /索引名/~~类型名~~/文档的ID
{请求体}

一般不使用类型名，推荐的是：`/{index}/_doc/{id}` 、 `/{index}/_doc` 和 `/{index}/_create/{id}`

> Elasticsearch 7 以后不推荐使用类型了，类型应该设置为 `_doc`

<u>如果要创建一个索引，并设定（映射/Mapping）该索引的字段类型（规则）</u>

可以这样（其中，test2 是 index 的名称）：

```http
PUT /test2
{
  "mappings": {
    "properties": {
      "name": {"type": "text"},
      "age": {"type": "long"},
      "bday": {"type": "date"}
    }
  }
}
```

如果不自己手动设置的话，ES 也会默认帮我们匹配合适的类型。

***

**GET**

`GET /test2` 获取名称为 test2 的 index 的信息

`GET _cat/health` ：查看 ES 健康信息

`GET _cat/indices?v`：查看索引的信息

***

**POST**

Elasticsearch 底层并不支持更新操作，所谓的更新，是将旧的文档删除，然后索引一个新的文档

可以这样写：

```http
POST /test2/_doc/1/_update
{
  "doc": {"name": "sally"}
}
```

更推荐的写法是 `/{index}/_update/{id}`：

```http
POST /test2/_update/1
{
  "doc": {"name": "john"}
}
```

***

**DELETE**

删除 test1 索引：`DELETE test1`

删除 test2 索引 ID 为 2 的文档：`DELETE test2/_doc/2`

## 基础查询操作

查询 test2 索引的全部数据：`GET test2/_search/`

基础的条件查询（三种）：

```http
GET test2/_doc/_search/?q=name:john

GET test2/_search/?q=name:john

GET test2/_search/
{
  "query": {
    "match": {
      "name": "john"
    }
  }
}
```

查询出来的结果里面有一行：`"_score" : 0.2876821`，这是匹配度/权重的意思，匹配度/权重越高，代表结果越匹配。

如果查询出来有多个结果，可以查看 max_score 的分值。

***

<u>多条件查询</u>

关键字匹配可以这样：

```http
GET test2/_search/
{
  "query": {
    "match": {
      "tags": "男 宅"
    }
  }
}
```

这里可以用空格隔开，表示只要包含"男"和"宅"关键字的，都要查出来。如果数据中有"男"这个字的，比如"宅男"、"男偶像"和"男人"都会被匹配出来，然后，有"宅"这个字的"宅男"也会被查出来，而且"宅男"的匹配度会更高，因为两个关键字都被匹配了

***

多个条件都要满足的时候，使用 bool 的 must（相当于 and）：

```http
GET test2/_search/
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {"name": "john"}
        },
        {
          "match": {"age": }
        }
      ]
    }
  }
}
```

只需要满足其中一个条件的时候，使用 should（相当于 or）：

```http
GET test2/_search/
{
  "query": {
    "bool": {
      "should": [
        {
          "match": {"name": "john"}
        },
        {
          "match": {"name": "wang"}
        }
        
      ]
    }
  }
}
```

多个条件都要排除的时候，使用 must_not：

```http
GET test2/_search/
{
  "query": {
    "bool": {
      "must_not": [
        {
          "match": {"name": "john"}
        },
        {
          "match": {"name": "wang"}
        }
      ]
    }
  }
}
```

如果还要过滤范围，可以使用 bool 的 filter。

假设要查询名字叫 John，年龄大于等于 6 岁，小于等于 15 岁的数据：

```http
GET test2/_search/
{
  "query": {
    "bool": {
      "must": [{
          "match": {"name": "john"}
        }],
      "filter": [{
        "range": {
          "age": {
            "gte": 6,
            "lte": 15
          }
        }
      }]
    }
  }
}
```

这里的 "gte" 表示 greater than or equal to，"lte" 表示 less than or equal to

还可以使用 gt 表示大于，lt 表示小于，范围也可以只写一个，比如 "gt": 6 就是大于 6（不写小于多少）

***

<u>过滤结果</u>

在查询结果的 `hits` 中，可以看到 `_source` 属性，我们可以通过指定这个属性，来让结果只展示部分内容。

类似于 MySQL 的 `select name, age from student;` 只会展示 student 表中的 name 和 age，不会展示其他的内容。

```http
GET test2/_search/
{
  "query": {
    "match": {
      "name": "john"
    }
  },
  "_source": ["name", "age"]
}
```

***

<u>排序</u>

使用 "sort"，

```http
GET test2/_search/
{
  "query": {
    "match": {
      "name": "john"
    }
  },
  "sort": [
    {
      "age": {
        "order": "desc"
      }
    }
  ]
}
```

***

<u>分页</u>

```http
GET test2/_search/
{
  "query": {
    "match": {
      "name": "john"
    }
  },
  "from": 1,
  "size": 4
}
```

相当于 SQL 语句 `select * from test2 where name = "john" limit 1, 4；`

## 精确查询、模糊查询和高亮查询

ES 字符串类型中，text 和 keyword 的区别：

* text：会分词，然后进行索引
* keyword：不会被分词器解析，直接索引

***

term（条款/条件）查询是直接通过 inverted index 精确查询，不会再对搜索词进行分词，一般对 keyword 类型使用

```http
GET /customer/doc/_search/
{
  "query": {
    "term": {
      "title": "blog"
    }
  }
}
```

而 terms 可以查询某个字段里含有多个关键词的文档：

```http
GET /customer/doc/_search/
{
  "query": {
    "terms": {
      "title":  ["blog","first"]
    }
  }
}
```

***

如果要实现高亮，可以这样：

```http
GET test2/_search
{
  "query": {
    "match": {"name": "john"}
  },
  "highlight": {
    "fields": {
      "name": {}
    }
  }
}
```

返回的结果里面有：

```http
"highlight" : {
  "name" : [
    "<em>john</em>"
  ]
}
```

这个 em 标签就是高亮标签。如果要自定义标签的话，可以利用 pre_tags 和 post_tags：

```http
GET test2/_search
{
  "query": {
    "match": {"name": "john"}
  },
  "highlight": {
    "pre_tags": "<p style='color:red'>", 
    "post_tags": "</p>", 
    "fields": {
      "name": {}
    }
  }
}
```

***

match 能够通过分词器来模糊查询（如果不设置，就使用默认的分词器）

设置分词器的方法：

假设有 index 为 account，类型为 person（类型的部分可以直接删掉并不会有影响），字段有 user、title 和 desc，我们可以在创建的时候这样设置：

```http
PUT /account
{
  "mappings": {
    "person": {
      "properties": {
        "user": {
          "type": "text",
          "analyzer": "ik_max_word",
          "search_analyzer": "ik_max_word"
        },
        "title": {
          "type": "text",
          "analyzer": "ik_max_word",
          "search_analyzer": "ik_max_word"
        },
        "desc": {
          "type": "text",
          "analyzer": "ik_max_word",
          "search_analyzer": "ik_max_word"
        }
      }
    }
  }
}
```

也可以这样设置 index 为 test3 的默认分词器：

```http
PUT test3
{
  "settings": {
    "analysis": {
      "analyzer": {
        "ik":{
          "tokenizer":"ik_max_word"
        }
      }
    }
  }
}
```

# Spring Boot 整合 Elasticsearch 

[官方文档：Java REST Client [master] » Java High Level REST Client » Search APIs » Search API](https://www.elastic.co/guide/en/elasticsearch/client/java-rest/master/java-rest-high-search.html)

参考：

* [SpringBootからElasticsearchを使う方法2選(クライアント/インデックス生成)](https://zenn.dev/pakio/articles/be541633cf23d260819b)
* [SpringBootからElasticsearchを使う方法2選 (CREATE/READ編)](https://zenn.dev/pakio/articles/52be822c1341a3ffacb9)
* [ElasticsearchでもSQL書きたい！ Opendistro for Elasticsearch SQL入門](https://zenn.dev/pakio/articles/aa7d9c12616cd4219503)
* [Elasticsearch SQL入門](https://zenn.dev/pakio/articles/elasticsearch-sql)
* [一、SpringBoot集成Elasticsearch7.4 实战（一）](https://www.jianshu.com/p/1fbfde2aefa5)
* [二、SpringBoot集成Elasticsearch7.4 实战（二）](https://www.jianshu.com/p/acc8e86cc772)
* [三、SpringBoot集成Elasticsearch7.4 实战（三）](https://www.jianshu.com/p/c02e5b412675)
* [SpringBoot集成Elasticsearch7.4 的 GitHub](https://github.com/king-angmar/weathertop/tree/master/akkad-springboot/springboot-elasticsearch)

使用 IDEA 创建空项目的话：

1. 记得勾选基本的依赖，以及在 NoSQL 里面找到 Elasticsearch
2. 在 Project Structure 里面的
    1. Project 里面把 SDK 该为现在使用的 JDK 版本，把 Project Language Level 改成和当前 JDK 相同的版本
    2. Modules 里面的 Language Level 改成 8 版本（或者 8 以上，不需要和 JDK 版本相同）
3. 在设置里面，把 Java Complier（关键字 javac）里面的 Target bytecode version 改成 1.8 或以上的版本（不需要和 JDK 版本相同）
4. 设置里面的 Languages & Frameworks 里面的 JavaScript 至少要是 ECMAScript 6

> API 的使用笔记在 [GitHub](https://github.com/LearnDifferent/es-api-test) 上

源码分析：

源码在两个地方。

主要目录是：

> /.m2/repository/org/springframework/boot/spring-boot-autoconfigure/2.3.7.RELEASE/spring-boot-autoconfigure-2.3.7.RELEASE.jar!/org/springframework/boot/autoconfigure

也就是在 IDEA 里找到 Maven 的 spring-boot-autoconfigure 里面的 org.springframework.boot.autoconfigure

在 autoconfigure 文件夹底下的 data 文件夹（存放数据/数据库相关）里面有一个 elasticsearch 文件夹。这个 elasticsearch 文件夹里面有 ElasticsearchDataAutoConfiguration 类。

而在 autoconfigure 文件夹里面也有 elasticsearch 文件夹。

这个 elasticsearch 文件夹底下有 ElasticsearchRestClientAutoConfiguration 类。

这个类导入了另外 3 个类：@Import({RestClientBuilderConfiguration.class, RestHighLevelClientConfiguration.class, RestClientFallbackConfiguration.class})

主要就看这导入的类，也就是源码提供的对象：

* RestClientBuilderConfiguration
* RestHighLevelClientConfiguration
* RestClientFallbackConfiguration

这 3 个类都是 ElasticsearchRestClientConfigurations 的内部静态类，最主要的是 RestHighLevelClientConfiguration 类

# Elasticsearch 集群

## ES 集群基础概念

集群：

* Elasticsearch 就算只有一个节点（Node），也被称为集群（Cluster）。一个集群中的节点，没有上限。
* 同一个集群有唯一的名字，默认是「elasticsearch」。一个节点只能通过某个集群的名字，来加入这个集群。
* 每个节点也有自己的名字，默认是使用漫威里面的随机角色来命名。
* 如果启动多个节点的时候，没有专门设置从属于哪个集群，那么所有启动了的节点，都会加入默认的「elasticsearch」集群中。
* ES 集群无法指定 Master 和 Slave，它会自己自动选举 Master

分片（Shards）

* ES 会把索引（indices）分为多个分片（shards），根据算法分配到所有节点（nodes）中
* 每个分片可以看成独立的索引，在查询的时候，会从分片中获取需要的内容，然后汇聚在一起，返回最终结果

复制（Replicas）

* 「复制」是分片（shards）的复制（replicas）
* 每个节点都有属于自己的分片（用于搜索），以及除了自己的分片之外的其他分片的复制/副本（用于备份）
    * 假设有 a b c 三个节点，索引被分为了 1 2 3 4 5 共五个分片
    * 那么 a 节点可能有 1 和 2 的分片，除此之外还有 3 和 4 的分片的复制（replica）
    * b 就可有可能是分片 3 和 4，然后保留了分配 1 和 5 的复制/副本
    * c 就被分配到了分片 5，保留了分配 2 的复制数据
* 也就是说，复制分片从不与原/主要（original/primary）分片放置于同一节点上

分片和复制设置：

* settings 中的 number_of_shards 和 number_of_replicas 可以设置
* 创建之后，只能更改 number_of_replicas 的数量，不能更改 number_of_shards
* ES 7 的默认情况下，分片和复制都是 1，也就是说，就算只有 1 个节点，也会在这个节点上备份分片

## 搭建 ES 集群

步骤：

1. 基础起步
    1. 一台服务器的情况下，使用 CP 指令，将 ES 的目录拷贝多份
    2. 如果之前有数据，注意把 ES 目录下的 data 目录里面的数据删掉
    3. 如果内存不够，在 ES 目录下使用 `vim config/jvm.options` 修改为 `-Xms256m`（或者更高）
2. 修改 ES 目录下的配置 `vim config/elasticsearch.yml`：
    * `cluster.name: [自定义集群名称]`
    * `node.name: [自定义该节点名称]`
    * `network.host: 0.0.0.0`：必须是 0.0.0.0，而且服务器要开启远程权限并关闭防火墙
    * `http.port: 9200`：如果是在一台服务器上，记得修改端口
    * `discovery.seed_hosts: ["IP:端口", "IP:端口"]`
        * 其他节点的 IP（域名）和 端口
        * 也可以直接只写 IP，需要注意的是：If the port is not given then it is determined by checking the following settings in order:
            1. transport.profiles.default.port
            2. transport.port
        * IPv6 addresses must be enclosed in square brackets
        * If a host name resolves via DNS to multiple addresses, Elasticsearch uses all of them.
        * 在之前的版本中叫做 discovery.zen.ping.unicast.hosts
        * For more information, consult the discovery and cluster formation module documentation，也就是官方的这篇：[Discovery and cluster formation settings](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-discovery-settings.html)
    * `gateway.recover_after_nodes: [数值]`：
        * Recover as long as this many data or master nodes have joined the cluster. 
        * 假设设置为 3，那么当集群内只有 1 个节点的时候，这 1 个节点会有全部的分片（没有备份）；当恢复到 2 个节点时候，这 2 个节点都有全部的分片（没有备份）；当恢复到 3 个节点的时候，就会开始恢复备份的情况
        * 具体参考[elasticsearch(es) 集群恢复触发配置（Local Gateway参数）](https://cloud.tencent.com/developer/article/1337490) 或用[印象笔记打开](https://app.yinxiang.com/shard/s72/nl/16998849/9445bc12-5874-43b4-83b1-b166745b0180/)
        * Deprecated in 7.7.0. This setting will be removed in 8.0. **Use `gateway.recover_after_data_nodes` instead**.
        * For more information, consult the gateway module documentation，也就是官方的这篇：[Local gateway settings](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-gateway.html)
    * 如果是在一台机器上搭载集群，要设置 `transport.port: [监听的端口]` （在一些低版本里是 transport.tcp.port），注意监听端口也不能被占用，默认是 9300 到 9400 之间

在任意一个节点上，使用 `GET _cat/health?v` 就可以查看整个集群的健康状况。

使用 Java 的 RestHighLevelClient 操作集群的时候，只要在配置的时候，把所有集群的 IP 和端口在 Builder 内通过 new HttpHost() 实例的形式创建出来就可以了。其他操作和之前一样。
