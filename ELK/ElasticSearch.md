[TOC]

## ES介绍

### 核心概念

#### 1、集群（Cluster）

一个或多个安装了 es 节点的服务器组织在一起，就是集群， 这些节点共同持有数据，共同提供搜索服务。

一个集群有一个名字 (Clsuter Name)，是集群的唯一标识，默认集群的名称是 elasticsearch ，具有相同名称的节点才会组成一个集群。

可在 config/elasticsearch.yml 文件中配置集群名称：

```yml
cluster.name: my-application
```

在集群中，节点的状态有三种：绿色、黄色、红色 ：

- 绿色 ：节点运行状态为健康状态。 所有的主分片、副本分片都可以正常工作。

- 黄色： 表示节点的运行状态为警告状态，所有主分片目前都可以直接运行，但是至少有一个副本分片是不能正常工作的。
- 红色： 表示集群无法正常工作，

#### 2、节点（Node）

集群中的一个服务器就是一个节点，节点中的会存储数据，同时参与集群的索引以及搜索功能。一个节点想要加入一个集群，只需要配置一下集群名称即可。es默认的集群发现方式可能会发生 `脑裂现象` 。

#### 3、索引（Index）

- 名词：具有相似特征文档的集合。
- 动词：索引数据即数据进行索引操作。

#### 4、类型（Type）

类型是索引上的逻辑分类或者分区。在es6之前，一个索引中可以有多个类型，从es7开始，一个索引中，只能有一个类型。

#### 5、文档（Document）

一个可以被索引的数据单元。文档都是 JSON 格式的。

#### 6、分片（Shards）

索引都是存储在节点上的，但是受节点的空间大小以及数据处理能力，单个节点的处理效果可能不理想，此时可以对索引进行分片。当我们创建一个索引的时候，就需要指定分片的数量，每个分片本身也是一个功能完善并且独立的索引。

默认情况下，一个索引会自动创建1个分片，并且为每一个分片创建一个副本。

#### 7、副本（Replicas）

副本也就是备份，是对主分片的一个备份。

#### 8、Settings

集群中对索引的定义信息。

#### 9、Mapping

Mapping 保存了定义索引字段的存储类型、分词方式、是否存储等信息。

#### 10、Analyzer

字段分词器的定义。

### ElasticSearch VS 关系型数据库

| 关系型数据库         | ElasticSearch                  |
| -------------------- | ------------------------------ |
| 数据库               | 索引                           |
| 表                   | 类型                           |
| 行                   | 文档                           |
| 列                   | 字段                           |
| 表结构               | 映射（Mapping）                |
| SQL                  | DSL（Domain Specfic Language） |
| selct * from xxx     | GET http://                    |
| update xx set xx=xxx | PUT http://                    |
| delete xx            | DELETE http://                 |
| 索引                 | 全文索引                       |

## ES基本使用

### 分词器

ElasticSearch 核心功能就是数据检索。查询分析主要由两步：

1. 词条化：**分词器**将输入的文本转为一个一个的词条流。

2. 过滤：比如停用词过滤器会从词条中去除不相干的词条（的，嗯，啊，呢）；另外还有同义词过滤器，小写过滤器等。

#### 内置分词器

| 分词器                                                       | 作用                                                       |
| ------------------------------------------------------------ | ---------------------------------------------------------- |
| [Standard Analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-standard-analyzer.html) | 标准分词器，适用于英语等。                                 |
| [Simple Analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-simple-analyzer.html) | 简单分词器，基于非字母字符进行分词，单词会被转为小写字母。 |
| [Whitespace Analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-whitespace-analyzer.html) | 空格分词器。按照空格进行切分。                             |
| [Stop Analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-stop-analyzer.html) | 类似于简单分词器，但是增加了停用词的功能。                 |
| [Keyword Analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-keyword-analyzer.html) | 关键词分词器，输入文本等于输出文本                         |
| [Pattern Analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-pattern-analyzer.html) | 利用正则表达式对文本进行切分，支持停用词。                 |
| [Language Analyzers](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-lang-analyzer.html) | 针对特定语言的分词器。                                     |
| [Fingerprint Analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-fingerprint-analyzer.html) | 指纹分词器，通过创建标记进行重复检测。                     |

#### 中文分词器

在 Es 中，适用较多的中文分词器是 [elasticsearch-analysis-ik](https://github.com/medcl/elasticsearch-analysis-ik) ，这是 es 的一个第三方插件。

#### 自定义扩展词库

##### 1、本地自定义

在 `es/plugins/ik/config` 目录下，新建 `ext.dic` 文件（文件名任意）在该文件中可以配置自定义的词库。如果有多个词，换行写入新词即可。

然后在 `es/plugins/ik/config/IKAnalyzer.cfg.xml` 中配置扩展词典的位置。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
<properties>
	<comment>IK Analyzer 扩展配置</comment>
	<!--用户可以在这里配置自己的扩展字典 -->
	<entry key="ext_dict"></entry>
	 <!--用户可以在这里配置自己的扩展停止词字典-->
	<entry key="ext_stopwords"></entry>
	<!--用户可以在这里配置远程扩展字典 -->
	<!-- <entry key="remote_ext_dict">words_location</entry> -->
	<!--用户可以在这里配置远程扩展停止词字典-->
	<!-- <entry key="remote_ext_stopwords">words_location</entry> -->
</properties>
```

##### 2、远程词库

也可以配置远程词库，远程词库支持热更新（不用重启 es 就可以生效）。

热更新只需要提供一个接口，接口返回扩展词即可。

热更新，主要是响应头的 `Last-Modified` 或者 `ETag` 字段发生变化， ik 就会自动重新加载远程扩展。

### 索引基本操作

#### [新建索引](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-create-index.html)

1、通过 head 插件新建索引。

2、通过请求创建。

创建索引请求：`PUT /<index>`

```apl
PUT book
```

创建成功后，可以查看索引信息：

```json
{
    "version":5,
    "mapping_version":1,
    "settings_version":1,
    "aliases_version":1,
    "routing_num_shards":1024,
    "state":"open",
    "settings":{
        "index":{
            "routing":{
                "allocation":{
                    "include":{
                        "_tier_preference":"data_content"
                    }
                }
            },
            "number_of_shards":"1",
            "provided_name":"book",
            "creation_date":"1627691823502",
            "number_of_replicas":"1",
            "uuid":"ui-vMRC1T8y8O1hito-D5g",
            "version":{
                "created":"7130499"
            }
        }
    },
    "mappings":{

    },
    "aliases":[

    ],
    "primary_terms":{
        "0":1
    },
    "in_sync_allocations":{
        "0":[
            "BgWVcwwdTbGmau4inkc_hw",
            "-eY3Rn-BTkWLEJ5cK_5Scw"
        ]
    },
    "rollover_info":{

    },
    "system":false,
    "timestamp_range":{
        "unknown":true
    }
}
```

创建索引时，也可以带上请求体和请求参数，如：aliases、mappings、settings 。

```json
PUT /my-index-000001
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 2
  }
}
```

> 需要注意两点：
>
> - 索引名称不能有大写字母。
> - 索引名是唯一的，不能重复，重复创建会出错。

#### 更新索引

如：修改索引的副本数

```json
PUT book/_settings
{
  "number_of_replicas": 2
}
```

修改索引的读写权限

```json
PUT /book/_settings
{
  "blocks.read": true,
  "blocks.write": true,
  "blocks.read_only": true,
}
```

#### [查看索引](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-get-index.html)

请求查看

```josn
GET /book
GET /book/_alias
GET book/_settings
```

也可以同时查看多个索引信息：

```josn
GET test,book/_settings
```

也可以查看所有索引信息：

```json
GET _all/_settings
```

#### [删除索引](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-delete-index.html)

```json
DELETE /my-index-000001
DELETE /my-index-000001/_alias/alias1
DELETE /_index_template/my-index-template
```

删除一个不存在的索引会报错。

#### 索引打开/关闭

[关闭索引](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-close.html) 

```json
POST /my-index-000001/_close
```

[打开索引](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-open-close.html)

```json
POST /my-index-000001/_open
```

#### [复制索引](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/docs-reindex.html)

索引复制，只会复制数据，不会复制索引配置。复制没有数据的索引，不会复制成功。

```json
POST _reindex
{
  "source": {"index": "book"},
  "dest": {"index": "book_new"}
}
```

#### [索引别名](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/indices-add-alias.html)

可以为索引创建别名，如果这个别名是唯一的，该别名可以代替索引名称。

```json
POST /book/_alias/mybook

POST /_aliases
{
  "actions": [
    {
      "add": {
        "index": "book",
        "alias": "alias1"
      }
    }
  ]
}
```

[删除索引别名](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/indices-delete-alias.html)。

```json
DELETE /book/_alias/mybook 

POST /_aliases
{
  "actions": [
    {
      "remove": {
        "index": "book",
        "alias": "alias1"
      }
    }
  ]
}
```

[查看索引别名](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/indices-get-alias.html)。

```json
// 查看某一个索引的别名
GET /book/_alias

// 查看某一个别名对应的索引
GET /alias1/_alias

// 查看集群上所有可用别名
GET /_alias
```



### 文档操作

#### [新建文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-index_.html)



```
PUT /<target>/_doc/<_id>

POST /<target>/_doc/

PUT /<target>/_create/<_id>

POST /<target>/_create/<_id>
```

如：

```
PUT blog/_doc/1
{
  "title": "myblog",
  "date": "2021-07-31",
  "content": "学习ES"
}
```

响应如下：

```
{
  "_index" : "blog",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 5,
  "result" : "created",
  "_shards" : {
    "total" : 2,
    "successful" : 2,
    "failed" : 0
  },
  "_seq_no" : 4,
  "_primary_term" : 1
}
```

- _index 表示文档索引
- _type 表示文档类型
- _id 表示文档的 id
- _version 表示文档的版本。（更新文档，版本会自动加1, 针对文档）
- result 表示执行结果
- _shards 表示分片信息
- _seq_no 和 _primary_term 是版本控制的（针对当前 index）

#### 获取文档

[获取文档](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/docs-get.html)

```
GET <index>/_doc/<_id>

HEAD <index>/_doc/<_id>

GET <index>/_source/<_id>

HEAD <index>/_source/<_id>
```

GET 用于获取文档，HEAD 用于探测文档是否存在。

[批量获取](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/docs-multi-get.html)

```
GET /_mget
GET /<index>/_mget
```



```json

GET blog/_mget
{
  "ids":["1"]
}

GET /_mget
{
  "docs":[
      {
        "_index": "blog",
        "_id":"1"
      }
    ]
}
```



#### [文档更新](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/docs-update.html)

获取文档

```
{
  "docs" : [
    {
      "_index" : "blog",
      "_type" : "_doc",
      "_id" : "1",
      "_version" : 6,
      "_seq_no" : 5,
      "_primary_term" : 1,
      "found" : true,
      "_source" : {
        "title" : "6666",
        "date" : "2021-07-31",
        "content" : "学习ES"
      }
    }
  ]
}
```

注意：文档更新一次，version 就会自增 1。



```
POST /<index>/_update/<_id>
```

这种方式，更新的文档会覆盖掉原文档，可以通过下面的脚本实现更新文档字段。

```json
POST /blog/_update/1
{
  "script": {
    "lang": "painless",
    "source": "ctx._source.title=params.title", 
    "params": {
      "title": "6666"
    }
  }
}
```

- lang 脚本语言， painless 是 es 内置的脚本语言
- ctx 是当前脚本语言执行对象的上下文，当前操作的文档对象，可通过 ctx 获取当前对象的属性，如 _source 、 _title 等。

也可以向文档中添加字段、修改数组内容等

```json
POST /blog/_update/1
{
  "script": {
    "lang": "painless",
    "source": "ctx._source.tags=[\"java\",\"php\"]"
  }
}

POST /blog/_update/1
{
  "script": {
    "lang": "painless",
    "source": "ctx._source.tags.add(\"js\")"
  }
}
```

也可以通过 if 构造稍微复杂一点的逻辑。

```json
POST /blog/_update/1
{
  "script": {
    "lang": "painless",
    "source": "if (ctx._source.tags.contains(\"123\")){ctx.op=\"delete\"}else{ctx.op=\"none\"}"
  }
}
```

#### [查询更新](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/docs-update-by-query.html)



通过条件查询到条件，然后再去更新。

```
POST /blog/_update_by_query
{
  "script": {
    "source": "ctx._source.content=\"888\"",
    "lang": "painless"
  },
 "query": {
   "term": {
     "title":"6666"
   }
 } 
}
```

将 title 为 6666 的文档的 title 更新为 888

#### 删除文档

[删除文档](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/docs-delete.html)

```
DELETE /<index>/_doc/<_id>
```

如果在添加文档时指定了路由， 则删除文档时也需要指定路由，否则删除失败

[查询删除](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/docs-delete-by-query.html)

```
POST /blog/_delete_by_query
{
  "query":{
    "match":{
      "title": "6666"
    }
  }
}

POST /book/_delete_by_query
{
  "query":{
    "term":{
      "title": "book"
    }
  }
}
```

删除一个索引下的所有文档

```
POST blog/_delete_by_query
{
  "query":{
    "match_all":{
      
    }
  }
}
```

#### 批量操作

es 中通过 [Bulk API](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/docs-bulk.html) 可以执行批量索引、批量删除、批量更新等操作。

首先需要将所有的批量操作写入一个 JSON 文件中，然后通过 POST 请求将该 JSON 文件上传并执行。

```json
POST _bulk
{"index":{"_index":"test", "_id":"1"}}
{"title":"hh"}
{"update":{"_index":"test", "_id":"1"}}
{"doc": {"title":"aa"}}
```

第一行：index 表示一个索引操作 （这个表示一个 action , 其他的 action 还有create、delete、update）。 _index 定义了索引名称， 这里表示创建一个名为 test 的索引， _id 表示新建文档的 id 为 1。

第二行是第一行操作的参数，表示将文档的 title 值为 hh，并更新为 aa 。最终获取到的结果：

```
{
  "_index" : "test1",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 2,
  "_seq_no" : 1,
  "_primary_term" : 1,
  "found" : true,
  "_source" : {
    "title" : "aa"
  }
}
```

### 文档路由

[查看文档被保存到哪个分片中](https://www.elastic.co/guide/en/elasticsearch/reference/current/cat-shards.html)

```
GET _cat/shards/blog?v
```

那么 es 中是按照什么规则去分配分片的？[routing-field](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-routing-field.html)

es 中的路由机制是通过哈希算法，将具有相同哈希值的文档放到一个主分片中，分片位置的计算方式如下：

> shard_num = hash(_routing) % num_primary_shards

routing 可以是任意一个字符串， es 默认是将文档的 id 作为 routing 值，通过哈希哈数根据 routing 生成一个数字，然后将改数字和分片数取余，取余的结果就是分片的位置。

默认的这种规则，最大的优势在于负载均衡，可以保证数据平均分配到不同的分片上，但有一个很大的劣势，就是查询的时候无法确定文档的位置，此时它会将请求广播到所有的的分片上去执行。另一方面，使用默认的路由模式，后期修改分片数量不方便。

开发者可以自定义 routing 的值，方式如下：

```
PUT test1/_doc/d?routing=ee
{
  "title":"ee"
}

GET test1/_doc/d?routing=ee
```

自定义 routing 可能会导致负载不均衡。

### 版本控制

在 es 中， 版本控制使用的锁是乐观锁。[ Optimistic concurrency control](https://www.elastic.co/guide/en/elasticsearch/reference/current/optimistic-concurrency-control.html)

#### es6.7 之前

在 es6.7 之前，使用 version + version_type 来进行乐观并发控制。



version 分为内部版本控制和外部版本控制。

##### 内部版本

es 自己维护的就是内部版本， 当创建一个文档时，  es 会给文档的版本赋值为 1。

每当用户修改一次文档，版本号就会自增 1。

如果使用内部版本， es 要求 version 参数的值必须和 es 文档中的 version 的值相等，才能操作成功。

##### 外部版本

在添加文档时，就指定版本号：

```
PUT book/_doc/b?version=100&version_type=external
{
  "title": "index"
}
```

以后更新的时候，版本要大于已有的版本号。

- version_type=external 或者 version_type=external_gte 表示以后更新的时候，版本要大于已有的版本号。
- version_type=external_gte 表示以后更新的时候，版本要大于等于已有的版本号。

#### 最新方案（ES6.7 之后）

现在使用 `if_seq_no` 和 `is_primary_term` 两个参数做并发控制。

```
PUT book/_doc/e?routing=ee
{
  "title":"ee"
}
```

结果：

```
{
  "_index" : "book",
  "_type" : "_doc",
  "_id" : "e",
  "_version" : 1,
  "result" : "created",
  "_shards" : {
    "total" : 3,
    "successful" : 3,
    "failed" : 0
  },
  "_seq_no" : 7,
  "_primary_term" : 1
}
```

`seq_no` 不属于某一个文档，它是属于整个索引的（version 则是属于某一个文档的，每个文档的 version 互不印象）。现在更新文档时，使用 `seq_no` 来做并发。由于 `seq_no` 是属于整个 index 的，所以任何文档的修改或者新增， `seq_no` 都会自增。

现在可以通过 `seq_no` 和 primary_term 来做乐观并发控制。

```
PUT blog/_doc/2?if_seq_no=5&if_primary_term=1
{
  "title":"6666"
}
```

### Elasticsearch 映射

#### 映射分类

##### 动态映射 

顾名思义，就是自动创建出来的映射，es 根据存入的文档，自动分析出来文档中字段的类型以及存储方式，这就是[动态映射](https://www.elastic.co/guide/en/elasticsearch/reference/current/dynamic-field-mapping.html)。



新建索引

```
PUT mappingtest
```

可以看到 Mappings 字段为空。

![image-20210801160735055](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/image-20210801160735055.png)

向索引中添加一个文档，文档添加成功后，会自动生成 Mappings。

![image-20210801161032873](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/image-20210801161032873.png)



可以看到，date 字段的类型为 date, title 的类型有两个， text 和 keyword 。（text 用于全文搜索， keyword  用于关键词搜索）

默认情况下，文档中如果新增了字段， Mappings 中也会自动新增进来。

如果新增字段时，希望能够抛出异常，可以通过 Mappings 中 [dynamic](https://www.elastic.co/guide/en/elasticsearch/reference/current/dynamic.html) 属性来配置。

dynamic 有四种取值：

- true ：默认即此，添加字段到 mapping
- runtime：新字段作为运行时字段添加到 mapping 中，这些字段未被索引，并在查询时从 _source 加载。
- false：忽略新字段，不会被索引和查询到，但是仍然会出现在 _source 中。
- strict：严格模式，发现新字段会抛出异常。

具体配置方式（其实就是静态映射）

```json
PUT blog
{
  "mappings": {
    "dynamic":"strict",
    "properties": {
      "title":{
        "type": "text"
      },
      "age":{
        "type": "long"
      }
    }
  }
}
```

然后向 blog 中添加文档：

```json
PUT blog/_doc/1
{
  "title":"test",
  "age":"15",
  "content":"what"
}
```

在添加的文档中，content 字段没有预定义， 添加操作返回报错：

```json
{
  "error" : {
    "root_cause" : [
      {
        "type" : "strict_dynamic_mapping_exception",
        "reason" : "mapping set to strict, dynamic introduction of [content] within [_doc] is not allowed"
      }
    ],
    "type" : "strict_dynamic_mapping_exception",
    "reason" : "mapping set to strict, dynamic introduction of [content] within [_doc] is not allowed"
  },
  "status" : 400
}
```

###### 动态映射字段类型映射

| **JSON data type**                                           | **`"dynamic":"true"`**                             | **`"dynamic":"runtime"`**                          |
| ------------------------------------------------------------ | -------------------------------------------------- | -------------------------------------------------- |
| `null`                                                       | No field added                                     | No field added                                     |
| `true` or `false`                                            | `boolean`                                          | `boolean`                                          |
| `double`                                                     | `float`                                            | `double`                                           |
| `integer`                                                    | `long`                                             | `long`                                             |
| `object`1                                                    | `object`                                           | `object`                                           |
| `array`                                                      | Depends on the first non-`null` value in the array | Depends on the first non-`null` value in the array |
| `string` that passes [date detection](https://www.elastic.co/guide/en/elasticsearch/reference/current/dynamic-field-mapping.html#date-detection) | `date`                                             | `date`                                             |
| `string` that passes [numeric detection](https://www.elastic.co/guide/en/elasticsearch/reference/current/dynamic-field-mapping.html#numeric-detection) | `float` or `long`                                  | `double` or `long`                                 |
| `string` that doesn’t pass `date` detection or `numeric` detection | `text` with a `.keyword` sub-field                 | `keyword`                                          |



如果一个被推断为日期类型或数字类型的字段被更新存储为其他类型的数据，则会报错。解决方式如下：

1、使用静态映射，在索引定义时，指定类型。

2、关闭日期检测和数字检测。

```json

PUT blog
{
  "mappings": {
    "date_detection": false, 
    "numeric_detection": false, 
  }
}
```

##### [静态映射（显示映射）](https://www.elastic.co/guide/en/elasticsearch/reference/current/explicit-mapping.html)



### [字段类型](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-types.html)



#### 核心数据类型

##### 字符串类型

- ~~string：（已过期），es5 之前用来描述字符串~~
- [text](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/text.html)：用来做全文检索，用了 text 后，字段内容会被分析， 在生成倒排索引之前，字符串会被分词器分成一个个词项。text 类型的字段不用于排序，很少用于聚合。这种字符串类型也被称为 analyzed 字段。
- [keyword](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/keyword.html) ：适用于结构化的字段，这种类型的字段可以用作过滤、排序、聚合等。也成为了 not-analyzed 字段。 

##### [数字类型](https://www.elastic.co/guide/en/elasticsearch/reference/current/number.html)

| 类型            | 描述                                                         |
| --------------- | ------------------------------------------------------------ |
| `long`          | A signed 64-bit integer with a minimum value of `-2^63` and a maximum value of `2^63-1`. |
| `integer`       | A signed 32-bit integer with a minimum value of `-2^31` and a maximum value of `2^31-1`. |
| `short`         | A signed 16-bit integer with a minimum value of `-32,768` and a maximum value of `32,767`. |
| `byte`          | A signed 8-bit integer with a minimum value of `-128` and a maximum value of `127`. |
| `double`        | A double-precision 64-bit IEEE 754 floating point number, restricted to finite values. |
| `float`         | A single-precision 32-bit IEEE 754 floating point number, restricted to finite values. |
| `half_float`    | A half-precision 16-bit IEEE 754 floating point number, restricted to finite values. |
| `scaled_float`  | A floating point number that is backed by a `long`, scaled by a fixed `double` scaling factor. |
| `unsigned_long` | An unsigned 64-bit integer with a minimum value of 0 and a maximum value of `2^64-1`. |

- 在满足需求的情况下，优先使用范围小的字段，字段的长度越短，索引和搜索的效率越高。

- 浮点数，优先使用 scaled_float （缩放浮点数）。 

```json
PUT product
{
  "mappings": {
    "properties": {
     "price":{
       "type": "scaled_float",
       "scaling_factor": 100
     }
    }
  }
}
```



##### [日期类型](https://www.elastic.co/guide/en/elasticsearch/reference/current/date.html)

由于 JSON 中没有日期类型，所以 es 中日期类型形式比较多样：

- 包含日期格式的字符串
- 自1970.1.1零点到现在的一个秒数或者毫秒数

es 内部将时间转为 UTC，然后将时间按照 milliseconds-since-the-epoch 的长整型来存储。

##### [布尔类型](https://www.elastic.co/guide/en/elasticsearch/reference/current/boolean.html)

| 值           | 说描述                                  |
| ------------ | --------------------------------------- |
| False values | `false`, `"false"`, `""` (empty string) |
| True values  | `true`, `"true"`                        |

##### [二进制类型](https://www.elastic.co/guide/en/elasticsearch/reference/current/binary.html)

二进制类型接收的是 Base64 编码的字符串，默认不存储，也不可搜索。

##### [范围类型](https://www.elastic.co/guide/en/elasticsearch/reference/current/range.html)

| 类型            | 描述                                                         |
| --------------- | ------------------------------------------------------------ |
| `integer_range` | A range of signed 32-bit integers with a minimum value of `-2^31` and maximum of `2^31-1`. |
| `float_range`   | A range of single-precision 32-bit IEEE 754 floating point values. |
| `long_range`    | A range of signed 64-bit integers with a minimum value of `-2^63` and maximum of `2^63-1`. |
| `double_range`  | A range of double-precision 64-bit IEEE 754 floating point values. |
| `date_range`    | A range of [`date`](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/date.html) values. Date ranges support various date formats through the [`format`](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/mapping-date-format.html) mapping parameter. Regardless of the format used, date values are parsed into an unsigned 64-bit integer representing milliseconds since the Unix epoch in UTC. Values containing the `now` [date math](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/common-options.html#date-math) expression are not supported. |
| `ip_range`      | A range of ip values supporting either [IPv4](https://en.wikipedia.org/wiki/IPv4) or [IPv6](https://en.wikipedia.org/wiki/IPv6) (or mixed) addresses. |



#### 复合类型

##### [数组类型](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/array.html)

在 Elasticsearch 中，没有专门的数组类型，默认情况下，任何字段都可以有0个或者多个值。但是，数组中的元素必须是同一种类型。

第一个添加进数组的元素的类型决定了整个数组的类型。

##### [对象类型](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/object.html)

由于 JSON 本身具有层级关系，所以文档包含内部对象，内部对象，还可以包含内部对象

##### [嵌套类型](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/nested.html)

nested 类型是 object 类型的特例。

如果使用 object 类型，假如有如下文档：

```JSON
PUT my-index-000001/_doc/1
{
  "group" : "fans",
  "user" : [ 
    {
      "first" : "John",
      "last" :  "Smith"
    },
    {
      "first" : "Alice",
      "last" :  "White"
    }
  ]
}
```

 由于 Lucene 没有内部对象的概念，因此，es 会将对象扁平化处理，将一个对象转化为字段名和值构成的简单列表。最终的文档

```json
{
  "group" :        "fans",
  "user.first" : [ "alice", "john" ],
  "user.last" :  [ "smith", "white" ]
}
```

扁平化之后，alice 和 white 的关联关系丢失了。此时，文档也能搜索到 alice simth。此时就可以使用 nested 类型来解决问题， nested 对象可以保持数组中每个对象的独立性。nested 类型将数组中的每个对象作为单独隐藏文档来索引，这样每一个嵌套对象都可以独立被索引。

```json
PUT my-index-000001
{
  "mappings": {
    "properties": {
      "user": {
        "type": "nested" 
      }
    }
  }
}

PUT my-index-000001/_doc/1
{
  "group" : "fans",
  "user" : [
    {
      "first" : "John",
      "last" :  "Smith"
    },
    {
      "first" : "Alice",
      "last" :  "White"
    }
  ]
}

GET my-index-000001/_search
{
  "query": {
    "nested": {
      "path": "user",
      "query": {
        "bool": {
          "must": [
            { "match": { "user.first": "Alice" }},
            { "match": { "user.last":  "Smith" }} 
          ]
        }
      }
    }
  }
}
```

优点：文档存储在一起，读取性能高。

缺点：更新父或者子文档时需要更新整个文档。

#### 地理类型

使用场景：

- 查找某一个范围内的地理位置
- 通过地理位置或者中心点的距离来聚合文档
- 把距离整合到文档的相关性分数中
- 按距离对文档进行排序

##### [Geo-point](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/geo-point.html)

此类型接受经纬度对，即坐标点。定义方式如下：

```json
PUT my-index-000001
{
  "mappings":{
    "properties": {
      "location":{
        "type": "geo_point"
      }
    }
  }
}
```

创建时指定字段类型，存储的时候，有五种方式：

```json
PUT my-index-000001/_doc/1
{
  "title":"Geo-point object",
  "location":{
    "lat": 41.12,
    "lon": -71.34
  }
}

PUT my-index-000001/_doc/2
{
  "title": "Geo-point String",
  "location":"42.12,-71.34"
}

PUT my-index-000001/_doc/3
{
  "title":"Geo-pointt GeoHash",
  "location":"drm3btev3e86"
}

PUT my-index-000001/_doc/4
{
  "title":"Geo-point Arrays",
  "location":[-71.34, 41.12]
}

PUT my-index-000001/_doc/5
{
  "text": "Geo-point as a WKT POINT primitive",
  "location" : "POINT (-71.34 41.12)" 
}


GET my-index-000001/_search
{
  "query": {
    "geo_bounding_box": { 
      "location": {
        "top_left": {
          "lat": 42,
          "lon": -72
        },
        "bottom_right": {
          "lat": 40,
          "lon": -74
        }
      }
    }
  }
}
```



##### [Geo-shape](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/geo-shape.html)

geo_shape 数据类型有助于对任意地理形状（例如矩形和多边形）进行索引和搜索。当被索引的数据或正在执行的查询包含除点以外的形状时，应该使用它。

形状可以使用 GeoJSON 或 WKT 格式表示。下表提供了 GeoJSON 和 WKT 到 Elasticsearch 类型的映射：

| GeoJSON Type         | WKT Type             | Elasticsearch Type   | Description                                                  |
| -------------------- | -------------------- | -------------------- | ------------------------------------------------------------ |
| `Point`              | `POINT`              | `point`              | A single geographic coordinate. Note: Elasticsearch uses WGS-84 coordinates only. |
| `LineString`         | `LINESTRING`         | `linestring`         | An arbitrary line given two or more points.                  |
| `Polygon`            | `POLYGON`            | `polygon`            | A *closed* polygon whose first and last point must match, thus requiring `n + 1` vertices to create an `n`-sided polygon and a minimum of `4` vertices. |
| `MultiPoint`         | `MULTIPOINT`         | `multipoint`         | An array of unconnected, but likely related points.          |
| `MultiLineString`    | `MULTILINESTRING`    | `multilinestring`    | An array of separate linestrings.                            |
| `MultiPolygon`       | `MULTIPOLYGON`       | `multipolygon`       | An array of separate polygons.                               |
| `GeometryCollection` | `GEOMETRYCOLLECTION` | `geometrycollection` | A GeoJSON shape similar to the `multi*` shapes except that multiple types can coexist (e.g., a Point and a LineString). |
| `N/A`                | `BBOX`               | `envelope`           | A bounding rectangle, or envelope, specified by specifying only the top left and bottom right points. |
| `N/A`                | `N/A`                | `circle`             | A circle specified by a center point and radius with units, which default to `METERS`. |

使用方法详见：[官方示例](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/geo-shape.html)

#### 特殊类型

##### [IP](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/ip.html) 

IP 类型可存储索引 IPV4 和 IPV6 地址。

```json
PUT my-index-000001
{
  "mappings": {
    "properties": {
      "ip_addr": {
        "type": "ip"
      }
    }
  }
}

PUT my-index-000001/_doc/1
{
  "ip_addr": "192.168.1.1"
}

GET my-index-000001/_search
{
  "query": {
    "term": {
      "ip_addr": "192.168.0.0/16"
    }
  }
}
```



##### [Token count](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/token-count.html)

用于统计字符串分词后的词项个数。

```json
PUT my-index-000001
{
  "mappings": {
    "properties": {
      "name": { 
        "type": "text",
        "fields": {
          "length": { 
            "type":     "token_count",
            "analyzer": "standard"
          }
        }
      }
    }
  }
}

PUT my-index-000001/_doc/1
{ "name": "John Smith" }

PUT my-index-000001/_doc/2
{ "name": "Rachel Alice Williams" }

GET my-index-000001/_search
{
  "query": {
    "term": {
      "name.length": 3 
    }
  }
}
```

### [Elasticsearch 映射参数](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/mapping-params.html)

#### [analyer](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/analyzer.html)

定义文本字段的分词器。默认对索引和查询都有效。

假设不用分词器，创建索引并添加文档后，查看词条向量（term vectors）

```json
GET my-index-000001/_termvectors/1
{
  "fields":["title"]
}
```

可以看到，默认情况下，中文就是一个一个字的分，这种分词没有意义。如果这样分词，查询只能按照一个字一个字来查

```json
GET my-index-000001/_search
{
  "query": {
    "term": {
      "title": "学"
    }
  }
}
```

所以，根据实际情况，要配置合适的分词器。

给字段设定分词器：

```json
PUT my-index-000001
{
  "mappings": {
    "properties": {
        "title":{
           "type": "text",
           "analyzer": "ik_smart"
      }
    }
  }
}
```

#### [search_analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/search-analyzer.html)

查询时候的分词器。默认情况下，如果没有配置 search_analyzer,则查询时，首先查看有没有 search_analyzer,有的话，就用 search_analyzer 来进行分词，如果没有，则看有没有 analyzer，如果有，则用 analyzer 来进行分词，否则使用 es 默认的分词器。

#### [normalizer](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/normalizer.html)

normalizer 参数用于解析前（索引或者查询）的标准化配置。

在 es 中，对于不想切分得字符串，通常会将其设置为 keyword，搜索的时候也是使用整个词进行搜索。如果在索引前没有做好数据清洗，导致大小写不一致，就会搜索不到，我们可以使用 normalizer 在索引之前以及查询之前进行文档的标准化。

如果使用了 normalizer，就可以在索引和查询时，分别对文档进行预处理。

normalizer 定义方式如下：

```json
PUT my-index-000001
{
  "settings": {
    "analysis": {
      "normalizer":{
        "my_normalizer":{
          "type":"custom",
          "filter":["lowercase"]
        }
      }
    }
  }, 
  "mappings": {
    "properties": {
        "author":{
           "type": "keyword",
           "normalizer": "my_normalizer"
      }
    }
  }
}
```

#### [boost](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/mapping-boost.html)

boost 参数可以设置字段的权重。

boost 有两种思路，一种是在定义 mappings 的时候使用，在指定字段类型时使用；另一种就是在查询时使用。

实际开发者建议使用后者，前者有问题：如果不重新索引文档，权重无法修改。

mapping 中使用 boost（不推荐）：

```json
PUT my-index-000001
{
  "mappings": {
    "properties": {
      "title":{
        "type": "text",
        "boost": 2
      }
    }
  }
}
```

另一种就是在查询时指定bootst:

```json
GET my-index-000001/_search
{
  "query": {
    "match": {
      "title":{
        "query":"学习",
        "boost": 2
      }
    }
  }
}
```



#### [coerce](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/mapping-boost.html)

coerce 用来清除脏数据，默认为 true。

例如数字用户可能写为 “99”，“99.0”。

通过 coerce 可以解决该问题。

如果需要修改 coerce ,方式如下：

```json
PUT my-index-000001
{
  "mappings": {
    "properties": {
      "age": {
        "type": "integer",
        "coerce": false
      }
    }
  }
}
```

此时，就不能向 age 字段传入字符串了。

#### [copy_to](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/copy-to.html)

这个属性，可以将多个字段的值，复制到同一个组字段中。

```json
PUT my-index-000001
{
  "mappings": {
    "properties": {
      "first_name": {
        "type": "text",
        "copy_to": "full_name" 
      },
      "last_name": {
        "type": "text",
        "copy_to": "full_name" 
      },
      "full_name": {
        "type": "text"
      }
    }
  }
}

PUT my-index-000001/_doc/1
{
  "first_name": "John",
  "last_name": "Smith"
}

GET my-index-000001/_search
{
  "query": {
    "match": {
      "full_name": { 
        "query": "John Smith",
        "operator": "and"
      }
    }
  }
}
```

#### [doc_values](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/doc-values.html) 和 fileddate

es 中的搜索主要是用倒排索引， doc_values 参数是为了加快排序、聚合操作而生的，当建立倒排索引的时候，会额外增加列式存储映射。

doc_values 默认是开启的，如果确定某个字段不需要排序或者不需要聚合，那么可以关闭 doc_values。

doc_values值是磁盘上的数据结构，在文档索引时构建，几乎所有字段类型都支持  值，但 text 和 annotated_text 字段除外。text 在查询时会生成一个 fileddata 的数据结构，fileddata 在字段首次被聚合、排序的时候生成。

| doc_values         | fielddata                      |
| ------------------ | ------------------------------ |
| 索引时创建         | 使用时动态创建                 |
| 磁盘               | 内存                           |
| 不占用内存         | 不占用磁盘                     |
| 索引速度稍微低一点 | 文档很多时，动态创建慢，占内存 |

doc_values 默认开启，fielddata 默认关闭。

```json
PUT users

PUT users/_doc/1
{
  "age":100
}

PUT users/_doc/2
{
  "age":99
}

PUT users/_doc/3
{
  "age":98
}

PUT users/_doc/4
{
  "age":101
}

GET users/_search
{
  "query": {
    "match_all": {}
  },
  "sort":[
    {
      "age":{
        "order": "desc"
      }
    }
    ]
}
```

由于 doc_values 默认时开启的，所以可以直接使用该字段排序，如果想关闭 doc_values ，如下：

```json
PUT my-index-000001
{
  "mappings": {
    "properties": {
      "status_code": { 
        "type":       "keyword"
      },
      "session_id": { 
        "type":       "keyword",
        "doc_values": false
      }
    }
  }
}
```

#### [dynamic](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/dynamic.html)

#### [enabled](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/enabled.html)

es 默认会索引所有的字段，但是有的字段可能只需要存储，不需要索引。此时可以通过 enabled 字段来控制。

```json
PUT blog
{
  "mappings": {
    "properties": {
      "title":{
        "enabled": false
      }
    }
  }
}
```

设置了 enabled 为 false 之后，就不可以再通过该字段进行搜索了。

#### [format](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/mapping-date-format.html)

日期格式。format 可以规范日期格式，而且一次可以定义多个 format。

```
PUT users
{
  "mappings": {
    "properties": {
      "birthday":{
        "type": "date",
        "format": "yyyy-MM-dd||yyyy-MM-dd HH:mm:ss"
      }
    }
  }
}
```

- 多个日期格式之间，使用 || 符号连接，注意没有空格
- 如果用户没有执行日期的 format ，默认的日期格式是 `strict_date_optional_time||epoch_mills`

#### [ignore_above](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/ignore-above.html)

长度超过 ignore_above 设置的字符串将不会被索引或存储。对于字符串数组，ignore_above 将分别应用于每个数组元素，并且不会索引或存储长于 ignore_above 的字符串元素。

#### [ignore_malformed](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/ignore-malformed.html)

默认情况下，尝试将错误的数据类型索引到字段中会引发异常，并拒绝整个文档。 ignore_malformed 参数如果设置为 true，则允许忽略异常。格式错误的字段没有被索引，但是文档中的其他字段被正常处理。

```json
PUT my-index-000001
{
  "mappings": {
    "properties": {
      "number_one": {
        "type": "integer",
        "ignore_malformed": true
      },
      "number_two": {
        "type": "integer"
      }
    }
  }
}

PUT my-index-000001/_doc/1
{
  "text":       "Some text value",
  "number_one": "foo" 
}

PUT my-index-000001/_doc/2
{
  "text":       "Some text value",
  "number_two": "foo" 
}
```

- document 1 : This document will have the `text` field indexed, but not the `number_one` field.
- document 2 : This document will be rejected because `number_two` does not allow malformed values.

#### ~~include_in_all~~

####  [index](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/mapping-index.html)

index 选项控制是否对字段值进行索引。它接受 true 或 false 并默认为 true。未编入索引的字段不可查询。

#### [index_options](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/index-options.html)

index_options 控制索引时哪些信息被存储到倒排索引中（用在 text 字段中），有四种取值：

**`docs`**

Only the doc number is indexed. Can answer the question *Does this term exist in this field?*

只存储文档编号

**`freqs`**

Doc number and term frequencies are indexed. Term frequencies are used to score repeated terms higher than single terms.

文档编号和词项频率被索引。重复词项的评分高于单个词项。

**`positions` (default)**

Doc number, term frequencies, and term positions (or order) are indexed. Positions can be used for [proximity or phrase queries](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/query-dsl-match-query-phrase.html).

文档编号、词项频率和词项位置（或顺序）被索引。位置可用于邻近或短语查询。

**`offsets`**

Doc number, term frequencies, positions, and start and end character offsets (which map the term back to the original string) are indexed. Offsets are used by the [unified highlighter](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/highlighting.html#unified-highlighter) to speed up highlighting.

文档编号、词项频率、词项位置、词项开始和结束的字符位置都被索引。偏移用来加速突出显示。

#### [norms](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/norms.html)

norms 对字段评分有用，text 默认开启 norms，如果不是特别需要，不要开启 norms。

```json
PUT my-index-000001/_mapping
{
  "properties": {
    "title": {
      "type": "text",
      "norms": false
    }
  }
}
```



#### [null_values](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/null-value.html)

在 es 中，值为 null 的字段不索引也不可以被搜索，null_value 可以让值为 null 的字段显式的可索引、可搜索：

```
PUT my-index-000001
{
  "mappings": {
    "properties": {
      "status_code": {
        "type":       "keyword",
        "null_value": "NULL" 
      }
    }
  }
}

PUT my-index-000001/_doc/1
{
  "status_code": null
}

PUT my-index-000001/_doc/2
{
  "status_code": [] 
}

GET my-index-000001/_search
{
  "query": {
    "term": {
      "status_code": "NULL" 
    }
  }
}
```



#### [position_increment_gap](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/position-increment-gap.html)

被解析的 text 字段会将 term 的位置考虑进去，以便能够支持邻近或短语查询。当索引具有多个值的文本字段时，会在值之间添加一个“假”间隙，以防止大多数短语查询跨值匹配。这个间隙的大小是使用 position_increment_gap 配置的，默认为 100。

```json
PUT my-index-000001/_doc/1
{
  "names": [ "John Abraham", "Lincoln Smith"]
}

GET my-index-000001/_search
{
  "query": {
    "match_phrase": {
      "names": {
        "query": "Abraham Lincoln" 
      }
    }
  }
}
// 可以通过 slop 指定空隙
GET my-index-000001/_search
{
  "query": {
    "match_phrase": {
      "names": {
        "query": "Abraham Lincoln",
        "slop": 101 
      }
    }
  }
}
// 也可在定义索引的时候，指定空隙
PUT my-index-000001
{
  "mappings": {
    "properties": {
      "names": {
        "type": "text",
        "position_increment_gap": 0 
      }
    }
  }
}

```



#### [properties](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/properties.html)

#### [similarity](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/similarity.html)

similarity 指定了文档的评分模型，默认有三种：

**`BM25`**

The [Okapi BM25 algorithm](https://en.wikipedia.org/wiki/Okapi_BM25). The algorithm used by default in Elasticsearch and Lucene.

**`classic`**

[7.0.0] Deprecated in 7.0.0.The [TF/IDF algorithm](https://en.wikipedia.org/wiki/Tf–idf), the former default in Elasticsearch and Lucene.

**`boolean`**

A simple boolean similarity, which is used when full-text ranking is not needed and the score should only be based on whether the query terms match or not. Boolean similarity gives terms a score equal to their query boost.

```
PUT my-index-000001
{
  "mappings": {
    "properties": {
      "default_field": { 
        "type": "text"
      },
      "boolean_sim_field": {
        "type": "text",
        "similarity": "boolean" 
      }
    }
  }
}
```



#### [store](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/mapping-store.html)

默认情况下，字段会被索引，也可以搜索，但是不会存储，虽然不会被存储的，但是 `_source` 中有一个字段的备份。如果想将字段存储下来，可以通过配置 store 来实现。

#### [term_vectors](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/term-vector.html)

term_vectors 是分词器产生的信息，包括

- 一组 terms
- 每个 term 的位置（或顺序）
- term 的首字符/尾字符与原始字符串原点的偏移量
- 有效载荷（如果可用）——与每个 term 位置相关的用户定义的二进制数据。



| term 取值                         | 描述                                                |
| --------------------------------- | --------------------------------------------------- |
| `no`                              | No term vectors are stored. (default)               |
| `yes`                             | Just the terms in the field are stored.             |
| `with_positions`                  | Terms and positions are stored.                     |
| `with_offsets`                    | Terms and character offsets are stored.             |
| `with_positions_offsets`          | Terms, positions, and character offsets are stored. |
| `with_positions_payloads`         | Terms, positions, and payloads are stored.          |
| `with_positions_offsets_payloads` | Terms, positions, offsets and payloads are stored.  |

#### [fields](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/multi-fields.html)

fields 参数可以让同一字段有多种不同的索引方式。 为了不同的目的以不同的方式索引相同的字段通常很有用。

例如，可以将字符串字段映射为用于全文搜索的文本字段，以及用于排序或聚合的关键字字段

```json
PUT my-index-000001
{
  "mappings": {
    "properties": {
      "city": {
        "type": "text",
        "fields": {
          "raw": { 
            "type":  "keyword"
          }
        }
      }
    }
  }
}

PUT my-index-000001/_doc/1
{
  "city": "New York"
}

PUT my-index-000001/_doc/2
{
  "city": "York"
}

GET my-index-000001/_search
{
  "query": {
    "match": {
      "city": "york" 
    }
  },
  "sort": {
    "city.raw": "asc" 
  },
  "aggs": {
    "Cities": {
      "terms": {
        "field": "city.raw" 
      }
    }
  }
}
```

### Elasticsearch 映射模板

es 中有动态映射，但是有的时候动态映射不能满足我们的需求，此时可以通过[映射模板](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/dynamic-templates.html)来解决。

```json
PUT my-index-000001
{
  "mappings": {
    "dynamic_templates": [
      {
        "longs_as_strings": {
          "match_mapping_type": "string",
          "match":   "long_*",
          "unmatch": "*_text",
          "mapping": {
            "type": "long"
          }
        }
      }
    ]
  }
}
```

### Elasticsearch 搜索入门

**搜索分为两个过程：**

**1、当向索引中保存文档时，默认情况下， es 中会保存两份内容， 一份是 `_source` 中的数据，另一份则是通过分词、排序等一系列过程生成的倒排索引文件，倒排索引中保存了词项和文档之间的对应关系。**

**2、搜索时，当 es 接收到用户的搜索请求之后，就会去倒排索引中查询，通过倒排索引中维护的倒排记录表找到关键词对应的文档集合，然后对文档进行评分、排序、高亮等处理，处理完成后返回文档。**

[Request body search | Elasticsearch Guide [7.13\] | Elastic](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/search-request-body.html)

```
PUT books
{
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "analyzer": "ik_max_word"
      },
      "publish":{
        "type": "text",
        "analyzer":"ik_max_word"
      },
      "type":{
        "type": "text",
        "analyzer": "ik_max_word"
      },
      "author":{
        "type": "keyword"
      },
      "info":{
        "type": "text",
        "analyzer": "ik_max_word"
      },
      "price":{
        "type": "double"
      }
    }
  }
}
```



#### [简单搜索](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/query-dsl-match-all-query.html)

查询文档：

```
GET books/_search
{
  "query": {
    "match_all": {}
  }
}
```

简写为

```
GET books/_search
```

简单搜索默认显示10 条记录。

#### [词项查询](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/query-dsl-term-query.html)

即 term 查询，就是根据 **词**去查询，查询指定字段中包含给定单词的文档， term 查询不被解析，只有搜索的词和文档中的词精确匹配，才会返回文档。应用场景如：人名、地名等。

```
GET books/_search
{
  "query": {
    "term": {
      "name": "十一五"
    }
  }
}
```

#### [分页](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/collapse-search-results.html)

```
GET books/_search
{
  "query": {
    "term": {
      "name": "十一五"
    }
  },
  "size": 10,
  "from": 10
}
```



#### [过滤返回字段](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/search-fields.html#source-filtering)

指定返回哪些字段

```
GET books/_search
{
  "query": {
    "term": {
      "name": "十一五"
    }
  },
  "size": 10,
  "from": 10,
  "_source": ["name", "author"]
}
```

#### [最小评分](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/search-search.html#search-api-min-score)

与关键词相关度低于指定分数的文档不返回，

```
GET books/_search
{
  "query": {
    "term": {
      "name": "十一五"
    }
  },
  "min_score":1.75,
  "_source": ["name","author"]
}
```

#### [高亮](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/highlighting.html)

查询关键字高亮：

```
GET books/_search
{
  "query": {
    "term": {
      "name": "十一五"
    }
  },
  "min_score":1.75,
  "_source": ["name","author"],
  "highlight": {
    "fields": {
      "name":{}
    }
  }
}
```

### Elasticsearch 全文查询

#### [match query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-query.html)

match query 会对查询语句进行分词，分词后，如果查询语句中的任何一个词项被匹配，则文档就会被索引到。

```
GET books/_search
{
  "query": {
    "match": {
      "name": "美术计算机"
    }
  }
}
```

这个查询首先会对 `美术计算机` 进行分词，分词之后，再去查询，只要文档中包含一个分词结果，就回返回文档。换句话说，默认词项之间是 OR 的关系，如果想要修改，也可以改为 AND。

```
GET books/_search
{
  "query": {
    "match": {
      "name": {
        "query": "美术计算机",
        "operator": "and"
      }
    }
  }
}
```

此时就回要求文档中必须同时包含 **美术** 和 **计算机** 两个词。

#### [match phrase query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-query-phrase.html)

match phrase query 也会对查询的关键字进行分词，但是它分词后有两个特点：

- 分词后的词项顺序必须和文档中词项的顺序一致
- 所有的词都必须出现在文档中

```
GET books/_search
{
  "query": {
    "match_phrase": {
        "name": {
          "query": "十一五计算机",
          "slop": 7
        }
    }
  }
}
```

query 是查询的关键字，会被分词器进行分解，分解之后去倒排索引中进行匹配。

slope 是指关键字之间的最小距离，默认是 0，但是注意不是关键字之间间隔的字数。文档中的字段被分词器解析之后，解析出来的词都包含一个 position字段表示词项的位置，查询短语分词之后的 position 之间的间隔要满足 slop 的要求。

![image-20210803212526853](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/image-20210803212526853.png)



#### [match phrase prefix query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-query-phrase-prefix.html)

这个类似于 match_phrase query，只不过这里多了一个通配符，match_phrase_prefix 支持最后一个词项的前缀匹配，但是这种匹配方式效率较低，了解即可。

```
GET books/_search
{
  "query": {
    "match_phrase_prefix": {
      "name": "计"
    }
  }
}
```

这个查询过程，会自动进行单词匹配，会自动查找以**计**开始的单词，默认返回 50 个词，可以自己控制：

```
GET books/_search
{
  "query": {
    "match_phrase_prefix": {
      "name": {
        "query": "计",
        "max_expansions": 3
      }
    }
  }
}
```

match_phrase_prefix 是针对分片级别的查询，假设 max_expansions 为 1，可能返回多个文档，但是只有一个词，这是我们预期的结果。有的时候实际返回结果和我们预期结果并不一致，原因在于这个查询是分片级别的，不同的分片确实只返回了一个词，但是结果可能来自不同的分片，所以最终会看到多个词。

#### [multi macth query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-multi-match-query.html)

match 查询的升级版，可以指定多个查询域：

```
GET books/_search
{
  "query": {
    "multi_match": {
      "query": "java",
      "fields": ["name","info"]
    }
  }
}
```

这种查询方式还可以指定字段的权重：

```
GET books/_search
{
  "query": {
    "multi_match": {
      "query": "阳光",
      "fields": ["name^4","info"]
    }
  }
}
```

这个表示关键字出现在 name 中的权重是出现在 info 中权重的 4 倍。

#### [query string query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-query-string-query.html)

query_string 是一种紧密结合 Lucene 的查询方式，在一个查询语句中可以用到 Lucene 的一些查询语法：

```
GET books/_search
{
  "query": {
    "query_string": {
      "default_field": "name",
      "query": "(十一五) AND (计算机)"
    }
  }
}
```



#### [simple_query_string](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-simple-query-string-query.html)

这个是 query_string 的升级，可以直接使用 +、|、- 代替 AND、OR、NOT 等。

```
GET books/_search
{
  "query": {
    "simple_query_string": {
      "fields": ["name"],
      "query": "(十一五) + (计算机)"
    }
  }
}
```



### Elasticsearch 词项查询

#### [term query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-term-query.html)

词项查询。词项查询不会分析查询字符，直接拿查询字符去倒排索引中比对。

#### [terms query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-terms-query.html)

词项查询，但是可以给定多个关键词。

```

GET books/_search
{
  "query": {
    "terms": {
      "name": ["程序","设计","java"]
    }
  }
}
```

#### [range query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-range-query.html)

范围查询，可以按照日期范围、数字范围等查询。

range query 中的参数主要有四个：

- gt
- lt
- gte
- lte

```
GET books/_search
{
  "query": {
    "range": {
      "price": {
        "gte": 10,
        "lt": 20
      }
    }
  },
  "sort": [
    {
      "price": {
        "order": "desc"
      }
    }
  ]
}
```

#### [exists query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-exists-query.html)

exists query 会返回指定字段中至少有一个`非空值`的文档：

```
GET books/_search
{
  "query": {
    "exists": {
      "field": "javaboy"
    }
  }
}
```

> **空字符串也是有值。null 是空值。**



#### [prefix query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-prefix-query.html)

前缀查询，效率略低，除非必要，一般不太建议使用。
给定关键词的前缀去查询：

```
GET books/_search
{
  "query": {
    "prefix": {
      "name": {
        "value": "大学"
      }
    }
  }
}
```

#### [wildcard query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-wildcard-query.html)

wildcard query 即通配符查询。支持单字符和多字符通配符：

- `?` 表示一个任意字符
- `*` 表示零个或者多个字符

```
GET books/_search
{
  "query": {
    "wildcard": {
      "author": {
        "value": "张*"
      }
    }
  }
}
```



#### [regexp query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-regexp-query.html)

支持正则表达式查询。

```
GET books/_search
{
  "query": {
    "regexp": {
      "author": "张."
    }
  }
}
```



#### [fuzzy query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-fuzzy-query.html)

在实际搜索中，有时我们可能会打错字，从而导致搜索不到，在 match query 中，可以通过 fuzziness 属性实现模糊查询。

fuzzy query 返回与搜索关键字相似的文档。怎么样就算相似？以 `LevenShtein` 编辑距离为准。编辑距离是指将一个字符变为另一个字符所需要更改字符的次数，更改主要包括四种：

- 更改字符（javb--〉java）
- 删除字符（javva--〉java）
- 插入字符（jaa--〉java）
- 转置字符（ajva--〉java）

为了找到相似的词，模糊查询会在指定的编辑距离中创建搜索关键词的所有可能变化或者扩展的集合，然后进行搜索匹配。

```
GET books/_search
{
  "query": {
    "fuzzy": {
      "name": "javba"
    }
  }
}
```



#### [ids query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-ids-query.html)

根据指定的 id 查询。

```
GET books/_search
{
  "query": {
    "ids":{
      "values":  [1,2,3]
    }
  }
}
```



### [Elasticsearch 复合查询](https://www.elastic.co/guide/en/elasticsearch/reference/current/compound-queries.html)

#### [constant score query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-constant-score-query.html)

当我们不关心检索词项的频率 （TF） 对搜索结果排序的影响时，可使用 constant_score 将查询语句或者过滤语句包裹起来。

```
GET books/_search
{
  "query": {
    "constant_score": {
      "filter": {
        "term": {
          "name": "java"
        }
      },
      "boost": 1.5
    }
  }
}
```

此结果的 _socre 都一样。

#### [bool query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-bool-query.html)

bool query 可以将任意多个查询组装在一起，有四种类型：

| Occur      | Description                                                  |
| ---------- | ------------------------------------------------------------ |
| `must`     | The clause (query) must appear in matching documents and will contribute to the score. |
| `filter`   | The clause (query) must appear in matching documents. However unlike `must` the score of the query will be ignored. Filter clauses are executed in [filter context](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-filter-context.html), meaning that scoring is ignored and clauses are considered for caching. |
| `should`   | The clause (query) should appear in the matching document.   |
| `must_not` | The clause (query) must not appear in the matching documents. Clauses are executed in [filter context](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-filter-context.html) meaning that scoring is ignored and clauses are considered for caching. Because scoring is ignored, a score of `0` for all documents is returned. |

- must ：文档必须匹配 must 选项下的查询条件
- filter: 类似于 must，但是 filter 不评分， 只是过滤数据
- should：文档应该匹配 should 下的查询条件，也可以不匹配
- must_not : 文档必须不满足 must_not 选项下的查询条件

bool 查询采用 *more-matches-is-better* 的方法，因此每个匹配 must 或 should 子句的分数将加在一起以提供每个文档的最终 _score。

```json
GET books/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "term": {
          "name": {
            "value": "java"
          }
        }}
      ],
      "must_not": [
        {
          "range": {
            "price": {
              "gte": 0,
              "lte": 35
            }
          }
        }
      ],
      "should": [
        {
          "match": {
            "info": "程序设计"
          }
        }
      ]
    }
  }
}
```

#### [minimum_should_match](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-minimum-should-match.html)

最小匹配度，可在 multi_match 和 should 查询中， 都可以设置 minimum_should_match 参数。

在查询时，文档会被分词，只要包含词项中的任意一项即会返回，在 match  查询中，可以通过 operator 参数设置文档必须匹配所有词项。

如果想匹配一部分词项，就要使用 minimum_should_match ，即至少匹配多少个词。

如下两个查询等价（参数 4 是因为查询关键字分词后有 4 项）：

```
GET books/_search
{
  "query": {
    "match": {
      "name": {
        "query": "语言程序设计",
        "minimum_should_match": 4
      }
    }
  }
}
GET books/_search
{
  "query": {
    "match": {
      "name": {
        "query": "语言程序设计",
        "operator": "and"
      }
    }
  }
}
```



#### [dis_max_query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-dis-max-query.html)

should query 的评分策略：

1. 首先会执行should 中的查询
2. 对查询结果的评分求和
3. 对求和结果乘以匹配语句总数
4. 将第三步的结果除以所有语句总数

dis_max query（disjunction max query，分离最大化查询）：匹配的文档依然返回，但是只将**最佳匹配的评分作为查询的评分**。

在 dis_max query 中，还有一个参数 `tie_breaker`（取值在0～1），在 dis_max query 中，是完全不考虑其他 query 的分数，只是将最佳匹配的字段的评分返回。但是，有的时候，我们又不得不考虑一下其他 query 的分数，此时，可以通过 `tie_breaker` 来优化 dis_max query。`tie_breaker` 会将其他 query 的分数，乘以 `tie_breaker`，然后和分数最高的 query 进行一个综合计算。

>If a document matches multiple clauses, the `dis_max` query calculates the relevance score for the document as follows:
>
>1. Take the relevance score from a matching clause with the highest score.
>2. Multiply the score from any other matching clauses by the `tie_breaker` value.
>3. Add the highest score to the multiplied scores.



#### [function_socre query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-function-score-query.html)

场景：例如想要搜索附近的肯德基，搜索的关键字是肯德基，但是希望将评分较高的肯德基优先展示，但是默认的评分策略是没有办法考虑到评分，只考虑相关性，这时候可通过 function_score query 实现。

测试数据

```
PUT blog/_doc/1
{
  "title":"Java集合详解",
  "votes":100
}

PUT blog/_doc/2
{
  "title":"Java多线程详解，Java锁详解",
  "votes":10
}
```

具体的思路，在旧的得分基础上，根据 votes 的数值进行综合运算，重新得出一个新的评分

评分有几种不同的计算方式：

- [`script_score`](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-function-score-query.html#function-script-score)
- [`weight`](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-function-score-query.html#function-weight)
- [`random_score`](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-function-score-query.html#function-random)
- [`field_value_factor`](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-function-score-query.html#function-field-value-factor)
- [decay functions](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-function-score-query.html#function-decay): `gauss`, `linear`, `exp`

##### weight

weight 可以对评分设置权重，就是在旧的评分基础上乘以 weight。

```
GET blog/_search
{
  "query": {
    "function_score": {
      "query": {
        "match": {
          "title": "java"
        }
      },
      "functions": [
        {
          "weight": 10
        }
      ]
    }
  }
}
```



##### rdndom_score

random_score 生成从 0 到但不包括 1 的均匀分布的分数。默认情况下，它使用内部 Lucene doc id 作为随机源，这非常有效，但遗憾的是无法重现。如果要重现，可以提供种子和字段。

```
GET blog/_search
{
  "query": {
    "function_score": {
      "query": {
        "match": {
          "title": "java"
        }
      },
      "functions": [
        {
          "random_score": {}
        }
      ]
    }
  }
}
```



##### script_score

自定义评分脚本。假设每个文档的最终得分是旧的分数加上votes。查询方式如下

```
GET blog/_search
{
  "query": {
    "function_score": {
      "query": {
        "match": {
          "title": "java"
        }
      },
      "functions": [
        {
          "script_score": {
            "script": {
              "lang": "painless",
              "source": "_score + doc['votes'].value"
            }
          }
        }
      ]
    }
  }
}	
```

现在，最终得分是 `(oldScore+votes)*oldScore`。

如果不想乘以 oldScore，查询方式如下：

```
GET blog/_search
{
  "query": {
    "function_score": {
      "query": {
        "match": {
          "title": "java"
        }
      },
      "functions": [
        {
          "script_score": {
            "script": {
              "lang": "painless",
              "source": "_score + doc['votes'].value"
            }
          }
        }
      ],
      "boost_mode": "replace"
    }
  }
}
```

boost_mode 参数取值：

| 参数值     | 参数含义                                                |
| ---------- | ------------------------------------------------------- |
| `multiply` | query score and function score is multiplied (default)  |
| `replace`  | only function score is used, the query score is ignored |
| `sum`      | query score and function score are added                |
| `avg`      | average                                                 |
| `max`      | max of query score and function score                   |
| `min`      | min of query score and function score                   |

##### **field_value_factor**

 `field_value_factor` 有几个数字参数选项：

| 参数       | 参数说明                                                     |
| ---------- | ------------------------------------------------------------ |
| `field`    | Field to be extracted from the document.                     |
| `factor`   | Optional factor to multiply the field value with, defaults to `1`. |
| `modifier` | Modifier to apply to the field value, can be one of: `none`, `log`, `log1p`, `log2p`, `ln`, `ln1p`, `ln2p`, `square`, `sqrt`, or `reciprocal`. Defaults to `none`. |

功能类似于 script_score，但是不用写脚本。

```
GET blog/_search
{
  "query": {
    "function_score": {
      "query": {
        "match": {
          "title": "java"
        }
      },
      "functions": [
        {
          "field_value_factor": {
            "field": "votes"
          }
        }
      ]
    }
  }
}
```

默认的得分就是`oldScore*votes`。

还可以加一个 modifer 进行更复杂的运算

```
GET blog/_search
{
  "query": {
    "function_score": {
      "query": {
        "match": {
          "title": "java"
        }
      },
      "functions": [
        {
          "field_value_factor": {
            "field": "votes",
            "modifier": "sqrt"
          }
        }
      ],
      "boost_mode": "replace"
    }
  }
}
```

此时，最终的得分是（sqrt(votes)）。

| Modifier     | Meaning                                                      |
| ------------ | ------------------------------------------------------------ |
| `none`       | Do not apply any multiplier to the field value               |
| `log`        | Take the [common logarithm](https://en.wikipedia.org/wiki/Common_logarithm) of the field value. Because this function will return a negative value and cause an error if used on values between 0 and 1, it is recommended to use `log1p` instead. |
| `log1p`      | Add 1 to the field value and take the common logarithm       |
| `log2p`      | Add 2 to the field value and take the common logarithm       |
| `ln`         | Take the [natural logarithm](https://en.wikipedia.org/wiki/Natural_logarithm) of the field value. Because this function will return a negative value and cause an error if used on values between 0 and 1, it is recommended to use `ln1p` instead. |
| `ln1p`       | Add 1 to the field value and take the natural logarithm      |
| `ln2p`       | Add 2 to the field value and take the natural logarithm      |
| `square`     | Square the field value (multiply it by itself)               |
| `sqrt`       | Take the [square root](https://en.wikipedia.org/wiki/Square_root) of the field value |
| `reciprocal` | [Reciprocate](https://en.wikipedia.org/wiki/Multiplicative_inverse) the field value, same as `1/x` where `x` is the field’s value |

还有个参数 factor ，影响因子。字段值先乘以影响因子，然后再进行计算。

还有一个参数 `max_boost`，控制计算结果的范围：

```
GET blog/_search
{
  "query": {
    "function_score": {
      "query": {
        "match": {
          "title": "java"
        }
      },
      "functions": [
        {
          "field_value_factor": {
            "field": "votes"
          }
        }
      ],
      "boost_mode": "sum",
      "max_boost": 100
    }
  }
}
```

`max_boost` 参数表示 **functions** 模块中，最终的计算结果上限。（不是整体的计算结果，仅表示 functions 模块的最终计算结果上限）如果超过上限，就按照上限计算。

#### [boosting query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-boosting-query.html)

boosting query 中包含三部分：

- positive：得分不变
- negative：降低得分
- negative_boost：降低的权重

```

GET books/_search
{
  "query": {
    "boosting": {
      "positive": {
        "match": {
          "name": "java"
        }
      },
      "negative": {
        "match": {
          "name": "2008"
        }
      },
      "negative_boost": 0.5
    }
  }
}
```



### [嵌套查询](https://www.elastic.co/guide/en/elasticsearch/reference/current/joining-queries.html)

#### 嵌套文档

```
PUT movies
{
  "mappings": {
    "properties": {
      "actors":{
        "type": "nested"
      }
    }
  }
}

PUT movies/_doc/1
{
  "name":"霸王别姬",
  "actors":[
    {
      "name":"张国荣",
      "gender":"男"
    },
    {
      "name":"巩俐",
      "gender":"女"
    }
    ]
}

```

##### 缺点：

查看文档数量

```
GET _cat/indices?v
```

![image-20210804201020254](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/image-20210804201020254.png)

发现只存了 1 个文档，但是查询出来是 3 个。这是因为 nested 文档在 es 内部其实也是独立的 lucene 文档，只是在我们查询的时候， es 内部帮我们做了 join 处理，所以最终看起来像一个独立文档，所以这种方案性能并不是特别好。而且，当我们更新子文档时，es 会重新索引整个文档。



#### [嵌套查询](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-nested-query.html)

```
GET movies/_search
{
  "query": {
    "nested": {
      "path": "actors",
      "query": {
        "bool": {
          "must": [
            {
              "match": {
                "actors.name": "张国荣"
              }
            },
            {
              "match": {
                "actors.gender": "男"
              }
            }
          ]
        }
      }
    }
  }
}
```



#### 父子文档

相比于嵌套文档，父子文档主要有如下优势：

- 更新父文档时，不会重新索引子文档
- 创建、修改或者删除子文档时，不会影响父文档或者其他的子文档。
- 子文档可以作为搜索结果独立返回。

如学生和班级的关系：

```
PUT stu_class
{
  "mappings": {
    "properties": {
      "name":{
        "type": "keyword"
      },
      "s_c":{
        "type": "join",
        "relations":{
          "class":"student"
        }
      }
    }
  }
}
```

`s_c` 表示父子文档关系的名字，可以自定义。join 表示这是一个父子文档。relations 里边，class 这个位置是 parent，student 这个位置是 child。

接下里，插入两个父文档：

```
PUT stu_class/_doc/1
{
  "name": "一班",
  "s_c":{
    "name": "class"
  }
}

PUT stu_class/_doc/2
{
  "name": "二班",
  "s_c":{
    "name": "class"
  }
}
```

再来添加三个子文档：

```
PUT stu_class/_doc/3?routing=1
{
  "name":"zhangsan",
  "s_c":{
    "name":"student",
    "parent":1
  }
}
PUT stu_class/_doc/4?routing=1
{
  "name":"lisi",
  "s_c":{
    "name":"student",
    "parent":1
  }
}
PUT stu_class/_doc/5?routing=2
{
  "name":"wangwu",
  "s_c":{
    "name":"student",
    "parent":2
  }
}
```

首先大家可以看到，子文档都是独立的文档。特别需要注意的地方是，子文档需要和父文档在同一个分片上，所以 routing 关键字的值为父文档的 id。另外，name 属性表明这是一个子文档。



> 父子文档需要注意的地方：
>
> 1. 每个索引只能定义一个 join filed
> 2. 父子文档需要在同一个分片上（查询，修改需要routing）
> 3. 可以向一个已经存在的 join filed 上新增关系



#### [has_child query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-has-child-query.html)

通过子文档查询父文档使用 `has_child` query。

```
GET stu_class/_search
{
  "query": {
    "has_child": {
      "type": "student",
      "query": {
        "match": {
          "name": "wangwu"
        }
      }
    }
  }
}
```



#### [has_parent query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-has-parent-query.html)

通过父文档查询子文档：

```
GET stu_class/_search
{
  "query": {
    "has_parent": {
      "parent_type": "class",
      "query": {
        "match": {
          "name": "二班"
        }
      }
    }
  }
}
```

查询二班的学生。但是大家注意，这种查询没有评分。

```
GET stu_class/_search
{
  "query": {
    "parent_id":{
      "type":"student",
      "id":1
    }
  }
}
```

通过 parent id 查询，默认情况下使用相关性计算分数。

#### 小结

整体上来说：

1. 普通子对象实现一对多，会损失子文档的边界，子对象之间的属性关系丢失。
2. nested 可以解决第 1 点的问题，但是 nested 有两个缺点：更新主文档的时候要全部更新，不支持子文档属于多个主文档。
3. 父子文档解决 1、2 点的问题，但是它主要适用于写多读少的场景。

### [Elasticsearch 地理位置查询](https://www.elastic.co/guide/en/elasticsearch/reference/current/geo-queries.html)

#### 数据准备

```
PUT geo
{
  "mappings": {
    "properties": {
      "name":{
        "type": "keyword"
      },
      "location":{
        "type": "geo_point"
      }
    }
  }
}
```

geo.json 文件



```
{"index":{"_index":"geo","_id":1}}
{"name":"西安","location":"34.288991865037524,108.9404296875"}
{"index":{"_index":"geo","_id":2}}
{"name":"北京","location":"39.926588421909436,116.43310546875"}
{"index":{"_index":"geo","_id":3}}
{"name":"上海","location":"31.240985378021307,121.53076171875"}
{"index":{"_index":"geo","_id":4}}
{"name":"天津","location":"39.13006024213511,117.20214843749999"}
{"index":{"_index":"geo","_id":5}}
{"name":"杭州","location":"30.259067203213018,120.21240234375001"}
{"index":{"_index":"geo","_id":6}}
{"name":"武汉","location":"30.581179257386985,114.3017578125"}
{"index":{"_index":"geo","_id":7}}
{"name":"合肥","location":"31.840232667909365,117.20214843749999"}
{"index":{"_index":"geo","_id":8}}
{"name":"重庆","location":"29.592565403314087,106.5673828125"}
```

导入

```
curl -XPOST "http://localhost:9200/geo/_bulk?pretty" -H "content-type:application/json" --data-binary @geo.json
```

工具网站：[geojson.io]([geojson.io](http://geojson.io/#map=9/36.6497/-243.2098))



#### [geo_distance query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-geo-distance-query.html)

给出一个中心点，查询距离该中心指定范围内的文档。

```

GET geo/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match_all": {}
        }
      ],
      "filter": [
        {
          "geo_distance": {
            "distance": "600km",
            "location": {
              "lat": 34.288991865037524,
              "lon": 108.9404296875
            }
          }
        }
      ]
    }
  }
}
```

以(34.288991865037524,108.9404296875) 为圆心，以 600KM 为半径，这个范围内的数据。

#### [geo_bounding_box query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-geo-bounding-box-query.html)

在某一个矩形内的点，通过两个点锁定一个矩形：

```
GET geo/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match_all": {}
        }
      ],
      "filter": [
        {
          "geo_bounding_box": {
            "location": {
              "top_left": {
                "lat": 32.0639555946604,
                "lon": 118.78967285156249
              },
              "bottom_right": {
                "lat": 29.98824461550903,
                "lon": 122.20642089843749
              }
            }
          }
        }
      ]
    }
  }
}
```



#### [geo_polygon query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-geo-polygon-query.html)

在某一个多边形范围内的查询。

```
GET geo/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match_all": {}
        }
      ],
      "filter": [
        {
          "geo_polygon": {
            "location": {
              "points": [
                {
                  "lat": 31.793755581217674,
                  "lon": 113.8238525390625
                },
                {
                  "lat": 30.007273923504556,
                  "lon":114.224853515625
                },
                {
                  "lat": 30.007273923504556,
                  "lon":114.8345947265625
                }
              ]
            }
          }
        }
      ]
    }
  }
}
```

给定多个点，由多个点组成的多边形中的数据。

#### [geo_shape query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-geo-shape-query.html)

`geo_shape` 用来查询图形。

新建索引：

```
PUT geo_shape
{
  "mappings": {
    "properties": {
      "name":{
        "type": "keyword"
      },
      "location":{
        "type": "geo_shape"
      }
    }
  }
}
```

然后添加一条线：

```
PUT geo_shape/_doc/1
{
  "name":"西安-郑州",
  "location":{
    "type":"linestring",
    "coordinates":[
      [108.9404296875,34.279914398549934],
      [113.66455078125,34.768691457552706]
      ]
  }
}
```

接下来查询某一个图形中是否包含该线：

```
GET geo_shape/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match_all": {}
        }
      ],
      "filter": [
        {
          "geo_shape": {
            "location": {
              "shape": {
                "type": "envelope",
                "coordinates": [
                  [
            106.5234375,
            36.80928470205937
          ],
          [
            115.33447265625,
            32.24997445586331
          ]
                ]
              },
              "relation": "within"
            }
          }
        }
      ]
    }
  }
}
```



[图形类型](https://www.elastic.co/guide/en/elasticsearch/reference/current/geo-shape.html#input-structure)

| GeoJSON Type         | WKT Type             | Elasticsearch Type   | Description                                                  |
| -------------------- | -------------------- | -------------------- | ------------------------------------------------------------ |
| `Point`              | `POINT`              | `point`              | A single geographic coordinate. Note: Elasticsearch uses WGS-84 coordinates only. |
| `LineString`         | `LINESTRING`         | `linestring`         | An arbitrary line given two or more points.                  |
| `Polygon`            | `POLYGON`            | `polygon`            | A *closed* polygon whose first and last point must match, thus requiring `n + 1` vertices to create an `n`-sided polygon and a minimum of `4` vertices. |
| `MultiPoint`         | `MULTIPOINT`         | `multipoint`         | An array of unconnected, but likely related points.          |
| `MultiLineString`    | `MULTILINESTRING`    | `multilinestring`    | An array of separate linestrings.                            |
| `MultiPolygon`       | `MULTIPOLYGON`       | `multipolygon`       | An array of separate polygons.                               |
| `GeometryCollection` | `GEOMETRYCOLLECTION` | `geometrycollection` | A GeoJSON shape similar to the `multi*` shapes except that multiple types can coexist (e.g., a Point and a LineString). |
| `N/A`                | `BBOX`               | `envelope`           | A bounding rectangle, or envelope, specified by specifying only the top left and bottom right points. |
| `N/A`                | `N/A`                | `circle`             | A circle specified by a center point and radius with units, which default to `METERS`. |



[relation](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-geo-shape-query.html#_spatial_relations) 属性表示两个图形的关系：

- `INTERSECTS` - (default) Return all documents whose `geo_shape` or `geo_point` field intersects the query geometry.
- `DISJOINT` - Return all documents whose `geo_shape` or `geo_point` field has nothing in common with the query geometry.
- `WITHIN` - Return all documents whose `geo_shape` or `geo_point` field is within the query geometry. Line geometries are not supported.
- `CONTAINS` - Return all documents whose `geo_shape` or `geo_point` field contains the query geometry.



### [Elasticsearch 特殊查询](https://www.elastic.co/guide/en/elasticsearch/reference/current/specialized-queries.html)

#### [more_like_this query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-mlt-query.html)

`more_like_this` query 可以实现基于内容的推荐，给定一篇文章，可以查询出和该文章相似的内容。

```
GET books/_search
{
  "query": {
    "more_like_this": {
      "fields": [
        "info"
      ],
      "like": "大学战略",
      "min_term_freq": 1,
      "max_query_terms": 12
    }
  }
}
```

- fields：要匹配的字段，可以有多个
- like：要匹配的文本
- min_term_freq：词项的最低频率，默认是 2。**特别注意，这个是指词项在要匹配的文本中的频率，而不是 es 文档中的频率**
- max_query_terms：query 中包含的最大词项数目
- min_doc_freq：最小的文档频率，搜索的词，至少在多少个文档中出现，少于指定数目，该词会被忽略
- max_doc_freq：最大文档频率
- analyzer：分词器，默认使用字段的分词器
- stop_words：停用词列表
- minmum_should_match



#### [script query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-script-query.html)

脚本查询



#### [percolate query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-percolate-query.html)

percolate query 译作渗透查询或者反向查询。

- 正常操作：根据查询语句找到对应的文档 query->document
- percolate query：根据文档，返回与之匹配的查询语句，document->query

应用场景：

- 价格监控
- 库存报警
- 股票警告
- ...

例如阈值告警，假设指定字段值大于阈值，报警提示。

percolate mapping 定义：

```
PUT log
{
  "mappings": {
    "properties": {
      "threshold":{
        "type": "long"
      },
      "count":{
        "type": "long"
      },
      "query":{
        "type":"percolator"
      }
    }
  }
}
```

percolator 类型相当于 keyword、long 以及 integer 等。

插入文档：

```
PUT log/_doc/1
{
  "threshold":10,
  "query":{
    "bool":{
      "must":{
        "range":{
          "count":{
            "gt":10
          }
        }
      }
    }
  }
}
```

最后查询：

```
GET log/_search
{
  "query": {
    "percolate": {
      "field": "query",
      "documents": [
        {
          "count":3
        },
        {
          "count":6
        },
        {
          "count":90
        },
        {
          "count":12
        },
        {
          "count":15
        }
        ]
    }
  }
}
```

查询结果中会列出不满足条件的文档。

查询结果中的 `_percolator_document_slot` 字段表示文档的 position，从 0 开始计。

```
{
  "took" : 23,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "log",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 1.0,
        "_source" : {
          "threshold" : 10,
          "query" : {
            "bool" : {
              "must" : {
                "range" : {
                  "count" : {
                    "gt" : 10
                  }
                }
              }
            }
          }
        },
        "fields" : {
          "_percolator_document_slot" : [
            2,
            3,
            4
          ]
        }
      }
    ]
  }
}

```

### Elasticsearch 搜索高亮与排序

#### [搜索高亮](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/highlighting.html)

普通高亮，默认会自动添加 em 标签：

```

GET books/_search
{
  "query": {
    "match": {
      "name": "大学"
    }
  },
  "highlight": {
    "fields": {
      "name": {}
    }
  }
}
```

可自定义高亮标签：

```
GET books/_search
{
  "query": {
    "match": {
      "name": "大学"
    }
  },
  "highlight": {
    "fields": {
      "name": {
        "pre_tags": ["<strong>"],
        "post_tags": ["</strong>"]
      }
    }
  }
}
```



#### [排序](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/sort-search-results.html)

排序很简单，默认是按照查询文档的相关度来排序的，即（`_score` 字段）：

```
GET books/_search
{
  "query": {
    "term": {
      "name": {
        "value": "java"
      }
    }
  }
}
```

等价于：

```
GET books/_search
{
  "query": {
    "term": {
      "name": {
        "value": "java"
      }
    }
  },
  "sort": [
    {
      "_score": {
        "order": "desc"
      }
    }
  ]
}
```

match_all 查询只是返回所有文档，不评分，默认按照添加顺序返回，可以通过 `_doc` 字段对其进行排序：

```
GET books/_search
{
  "query": {
    "match_all": {}
  },
  "sort": [
    {
      "_doc": {
        "order": "desc"
      }
    }
  ],
  "size": 20
}
```



### 指标聚合



#### [Max  Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/search-aggregations-metrics-max-aggregation.html)

```
GET books/_search
{
  "aggs": {
    "max_price": {
      "max": {
        "field": "price",
        "missing": 1000
      }
    }
  }
}
```

如果某个文档中缺少 price 字段，则设置该字段的值为 1000。

也可以通过脚本来查询最大值：

```
GET books/_search
{
  "aggs": {
    "max_price": {
      "max": {
        "script": {
          "source": "if(doc['price'].size()!=0){doc.price.value}"
        }
      }
    }
  }
}
```

使用脚本时，可以先通过 `doc['price'].size()!=0` 去判断文档是否有对应的属性。

#### [Min Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/search-aggregations-metrics-min-aggregation.html)

和Max 一致。



#### [Avg Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/search-aggregations-metrics-avg-aggregation.html)

略



#### [Cardinality Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/search-aggregations-metrics-cardinality-aggregation.html)

cardinality aggregation 用于基数统计。类似于 SQL 中的 distinct count(0)：

text 类型是分析型类型，默认是不允许进行聚合操作的，如果相对 text 类型进行聚合操作，需要设置其 fielddata 属性为 true，这种方式虽然可以使 text 类型进行聚合操作，但是无法满足精准聚合，如果需要精准聚合，可以设置字段的子域为 keyword。



#### [Stats Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/search-aggregations-metrics-stats-aggregation.html)

基本统计，一次性返回 count、max、min、avg、sum：

```
GET books/_search
{
  "aggs": {
    "stats_query": {
      "stats": {
        "field": "price"
      }
    }
  }
}
```



![image-20210805212713763](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/image-20210805212713763.png)





#### [Extends Stats Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/search-aggregations-metrics-extendedstats-aggregation.html)

高级统计，比 stats 多出来：平方和、方差、标准差、平均值加减两个标准差的区间：

```
GET books/_search
{
  "aggs": {
    "es": {
      "extended_stats": {
        "field": "price"
      }
    }
  }
}
```

![image-20210805212943517](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/image-20210805212943517.png)

#### [Percentiles Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/search-aggregations-metrics-percentile-aggregation.html)

百分位统计.

```
GET books/_search
{
  "aggs": {
    "p": {
      "percentiles": {
        "field": "price",
        "percents": [
          1,
          5,
          10,
          15,
          25,
          50,
          75,
          95,
          99
        ]
      }
    }
  }
}
```

结果

```
  "aggregations" : {
    "p" : {
      "values" : {
        "1.0" : 0.0,
        "5.0" : 0.0,
        "10.0" : 0.0,
        "15.0" : 13.479999999999995,
        "25.0" : 18.0,
        "50.0" : 28.0,
        "75.0" : 37.0,
        "95.0" : 62.19999999999982,
        "99.0" : 98.0
      }
    }
  }
```



#### [Value Count Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/search-aggregations-metrics-valuecount-aggregation.html)

可以按照字段统计文档数量（包含指定字段的文档数量）：

```
GET books/_search
{
  "aggs": {
    "count": {
      "value_count": {
        "field": "price"
      }
    }
  }
}
```



### [桶聚合](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/search-aggregations-bucket.html)



#### [Terms Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/search-aggregations-bucket-terms-aggregation.html)

分组聚合。统计不同出版社所出版的图书的平均价格：

```
GET books/_search
{
  "aggs": {
    "NAME": {
      "terms": {
        "field": "publish",
        "size": 20
      },
      "aggs": {
        "avg_price": {
          "avg": {
            "field": "price"
          }
        }
      }
    }
  }
}
```



#### [Filter Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/search-aggregations-bucket-filter-aggregation.html)

过滤器聚合。可以将符合过滤器中条件的文档分到一个桶中。

统计不同出版社所出版的图书的平均价格：

```
GET books/_search
{
  "aggs": {
    "NAME": {
      "filter": {
        "term": {
          "name": "java"
        }
      },
      "aggs": {
        "avg_price": {
          "avg": {
            "field": "price"
          }
        }
      }
    }
  }
}
```



#### [Filters Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/search-aggregations-bucket-filters-aggregation.html)

多过滤器聚合。过滤条件可以有多个。



#### [Range Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/search-aggregations-bucket-range-aggregation.html)

按照范围聚合，在某一个范围内的文档数统计。

```
GET books/_search
{
  "aggs": {
    "NAME": {
      "range": {
        "field": "price",
        "ranges": [
          {
            "to": 50
          },{
            "from": 50,
            "to": 100
          },{
            "from": 100,
            "to": 150
          },{
            "from": 150
          }
        ]
      }
    }
  }
}
```

此聚合包含每个范围的 from 值，但是不包含 to 值。

#### [Date Range Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/search-aggregations-bucket-daterange-aggregation.html)

Range Aggregation 也可以用来统计日期，但是也可以使用 Date Range Aggregation，后者的优势在于可以使用日期表达式。



统计一年前到一年后的博客数量：

```
GET blog/_search
{
  "aggs": {
    "NAME": {
      "date_range": {
        "field": "date",
        "ranges": [
          {
            "from": "now-12M/M",
            "to": "now+1y/y"
          }
        ]
      }
    }
  }
}
```

- 12M/M 表示 12 个月。
- 1y/y 表示 1年。
- d 表示天



#### [Date Histogram Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/search-aggregations-bucket-datehistogram-aggregation.html)

时间直方图聚合。

例如统计各个月份的博客数量

```
GET blog/_search
{
  "aggs": {
    "NAME": {
      "date_histogram": {
        "field": "date",
        "calendar_interval": "month"
      }
    }
  }
}
```



#### [Missing Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/search-aggregations-bucket-missing-aggregation.html)

空值聚合。

统计所有没有 price 字段的文档：

```
GET books/_search
{
  "aggs": {
    "NAME": {
      "missing": {
        "field": "price"
      }
    }
  }
}
```



#### [Chidlren Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/search-aggregations-bucket-children-aggregation.html)

可以根据父子文档关系进行分桶。

查询子类型为 student 的文档数量：

```
GET stu_class/_search
{
  "aggs": {
    "NAME": {
      "children": {
        "type": "student"
      }
    }
  }
}
```



#### [Geo Distance Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/search-aggregations-bucket-geodistance-aggregation.html)

对地理位置数据做统计。

例如查询(34.288991865037524,108.9404296875)坐标方圆 600KM 和 超过 600KM 的城市数量。

```
GET geo/_search
{
  "aggs": {
    "NAME": {
      "geo_distance": {
        "field": "location",
        "origin": "34.288991865037524,108.9404296875",
        "unit": "km", 
        "ranges": [
          {
            "to": 600
          },{
            "from": 600
          }
        ]
      }
    }
  }
}
```



#### [IP Range Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/search-aggregations-bucket-iprange-aggregation.html)

IP 地址范围查询。

```
GET blog/_search
{
  "aggs": {
    "NAME": {
      "ip_range": {
        "field": "ip",
        "ranges": [
          {
            "from": "127.0.0.5",
            "to": "127.0.0.11"
          }
        ]
      }
    }
  }
}
```



### [管道聚合](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/search-aggregations-pipeline.html)

管道聚合相当于在之前聚合的基础上，再次聚合。



#### [Avg Bucket Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/search-aggregations-pipeline-avg-bucket-aggregation.html)

计算聚合平均值。例如，统计每个出版社所出版图书的平均值，然后再统计所有出版社的平均值：

```
GET books/_search
{
  "aggs": {
    "book_count": {
      "terms": {
        "field": "publish",
        "size": 3
      },
      "aggs": {
        "book_avg": {
          "avg": {
            "field": "price"
          }
        }
      }
    },
    "avg_book":{
      "avg_bucket": {
        "buckets_path": "book_count>book_avg"
      }
    }
  }
}
```



#### [Max Bucket Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/search-aggregations-pipeline-max-bucket-aggregation.html)

统计每个出版社所出版图书的平均值，然后再统计平均值中的最大值：

```
GET books/_search
{
  "aggs": {
    "book_count": {
      "terms": {
        "field": "publish",
        "size": 3
      },
      "aggs": {
        "book_avg": {
          "avg": {
            "field": "price"
          }
        }
      }
    },
    "avg_book":{
      "max_bucket": {
        "buckets_path": "book_count>book_avg"
      }
    }
  }
}
```

#### [Min Bucket Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/search-aggregations-pipeline-min-bucket-aggregation.html)

#### [Sum Bucket Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/search-aggregations-pipeline-sum-bucket-aggregation.html)

#### [Stats Bucket Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/search-aggregations-pipeline-stats-bucket-aggregation.html)

统计每个出版社所出版图书的平均值，然后再统计平均值的各种数据：

```
GET books/_search
{
  "aggs": {
    "book_count": {
      "terms": {
        "field": "publish",
        "size": 3
      },
      "aggs": {
        "book_avg": {
          "avg": {
            "field": "price"
          }
        }
      }
    },
    "avg_book":{
      "stats_bucket": {
        "buckets_path": "book_count>book_avg"
      }
    }
  }
}

```



#### [Extend Stats Bucket Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/search-aggregations-pipeline-extended-stats-bucket-aggregation.html)

#### [Percentiles Bucket Aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/7.14/search-aggregations-pipeline-percentiles-bucket-aggregation.html)

