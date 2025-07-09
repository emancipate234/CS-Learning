#### 内容来自：Elasticsearch 权威指南

##### 1.elasticsearch如何执行分布式搜索

query then fetch

query 阶段：

![Query phase](./images/elas_0901.png)

1.客户端发送 search 请求到 Node 3 , Node 3 充当 coordinating Node , 会创建一个大小为 from + size 的空优先队列

2.Node 3 将请求转发到索引的每个主分片 primary shard 或者副本分片 replica shard 中。每个分片在本地执行查询并添加结果到大小为 from + size 的本地有序优先队列中

3.每个分片返回各自优先队列中所有文档的 id 和排序值给协调节点 coordinating Node , 也就是 Node 3 , 它合并这些值到自己的优先队列中来产生一个全局排序后的结果列表

fetch 阶段：

![Fetch Phase](./images/elas_0902.png)

1.协调节点辨别出哪些文档需要取回并向相关分片提交多个get请求

2.每个分片加载并丰富文档，如果有需要的话接着返回文档给协调节点

3.所有文档都被取回，协调节点返回结果给客户端

协调节点决定取回：例如指定  `{ "from": 90, "size": 10 }` ，最初的90个结果会被丢弃，只有从第91个开始的10个结果需要被取回

协调节点给持有相关文档的每个分片创建一个 [multi-get request](https://www.elastic.co/guide/cn/elasticsearch/guide/current/distrib-multi-doc.html) ，并发送请求给同样处理查询阶段的分片副本。

##### 2.elasticsearch分布式搜索中深分页问题

先查后取，每个分片创建from + size长度的队列，协调节点需要根据 number_of_shards * (from + size) 排序文档，来找到包含在 size 里的文档。当 from 值足够大，排序过程会非常沉重，“深分页” 很少符合人的行为。当2到3页过去以后，人会停止翻页

##### 3.偏好preference参数作用

参数 preference 允许用来控制由哪些分片或节点来处理搜索请求。接受 `_primary`, `_primary_first`, `_local`, `_only_node:xyz`, `_prefer_node:xyz`, 和 `_shards:2,3` 这样的值，具体参阅  [search `preference`](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/search-request-preference.html) 文档页面，这样可以避免 bouncing results 问题：

想象一下有两个文档有同样值的时间戳字段，搜索结果由 timestamp 字段排序。由于搜索请求在有效分片副本间轮询，可能发生主分片处理请求时，这两个文档是一个顺序，而副本分片处理请求时又是另一个顺序，用户刷新界面，搜索结果表现不同的顺序。可以设置 preference 参数为用户会话 id ，让用户始终使用同一个分片来解决

##### 使用路由

属于某用户的文档被存储在某个分片上，搜索的时候，指定几个routing值来限定只搜索几个相关的分片，与后续扩容有关 [*扩容设计*](https://www.elastic.co/guide/cn/elasticsearch/guide/current/scale.html) 

```json
GET /_search?routing=user_1,user2
```

##### 修改搜索类型

缺省的搜索类型是 `query_then_fetch` 。 在某些情况下，你可能想明确设置 `search_type` 为 `dfs_query_then_fetch` 来改善相关性精确度

```json
GET /_search?search_type=dfs_query_then_fetch
```

搜索类型 `dfs_query_then_fetch` 有预查询阶段，这个阶段可以从所有相关分片获取词频来计算全局词频  [被破坏的相关度！](https://www.elastic.co/guide/cn/elasticsearch/guide/current/relevance-is-broken.html) 会讨论

##### 4.如何不付出深度分页代价

`scroll` 查询 可以用来对 Elasticsearch 有效地执行大批量的文档查询，而又不用付出深度分页那种代价。允许我们先做查询初始化，再批量拉取结果。深度分页的代价根源是结果集全局排序，如果去掉全局排序的特性的话查询结果的成本就会很低。 游标查询用字段 `_doc` 来排序。 这个指令让 Elasticsearch 仅仅从还有结果的分片返回下一批结果。

游标查询会取某个时间点的快照数据，通过在查询的时候设置参数 `scroll` 的值为我们期望的游标查询的过期时间。 游标查询的过期时间会在每次做查询的时候刷新，所以这个时间只需要足够处理当前批的结果就可以了，而不是处理查询结果的所有文档的所需时间。设置这个超时能够让 Elasticsearch 在稍后空闲的时候自动释放这部分资源。

```json
GET /old_index/_search?scroll=1m 
{
    "query": { "match_all": {}},
    "sort" : ["_doc"], 
    "size":  1000
}
```

保持游标查询窗口1分钟，用关键字 _doc 排序

这个查询的返回结果包括一个字段 `_scroll_id`， 它是一个base64编码的长字符串 。 现在我们能传递字段 `_scroll_id` 到 `_search/scroll` 查询接口获取下一批结果

```json
GET /_search/scroll
{
    "scroll": "1m", 
    "scroll_id" : "cXVlcnlUaGVuRmV0Y2g7NTsxMDk5NDpkUmpiR2FjOFNhNnlCM1ZDMWpWYnRROzEwOTk1OmRSamJHYWM4U2E2eUIzVkMxalZidFE7MTA5OTM6ZFJqYkdhYzhTYTZ5QjNWQzFqVmJ0UTsxMTE5MDpBVUtwN2lxc1FLZV8yRGVjWlI2QUVBOzEwOTk2OmRSamJHYWM4U2E2eUIzVkMxalZidFE7MDs="
}
```

再次设置游标查询过期时间1分钟，这个游标查询返回下一批结果。

尽管我们指定字段 `size` 的值为1000，我们有可能取到超过这个值数量的文档。 当查询的时候， 字段 `size` 作用于单个分片，所以每个批次实际返回的文档数量最大为 `size * number_of_primary_shards`

注意游标查询每次返回一个新字段 `_scroll_id`。每次我们做下一次游标查询， 我们必须把前一次查询返回的字段 `_scroll_id` 传递进去。 当没有更多的结果返回的时候，我们就处理完所有匹配的文档了。

##### 手动创建索引

```json
PUT /my_index
{
    "settings": { ... any settings ... },
    "mappings": {
        "type_one": { ... any mappings ... },
        "type_two": { ... any mappings ... },
        ...
    }
}
```

禁止自动创建索引：config/elasticsearch.yml 的每个节点下添加：

```yaml
action.auto_create_index: false
```

用 [索引模板](https://www.elastic.co/guide/cn/elasticsearch/guide/current/index-templates.html) 来预配置开启自动创建索引。这在索引日志数据的时候尤其有用：你将日志数据索引在一个以日期结尾命名的索引上，子夜时分，一个预配置的新索引将会自动进行创建。

索引设置：number_of_shards 每个索引的主分片数，默认值5，配置在创建索引后不能修改

number_of_replicas 每个主分片的副本数，默认值1，对于活动的索引库可以随时修改

```json
PUT /my_temp_index
{
    "settings": {
        "number_of_shards" :   1,
        "number_of_replicas" : 0
    }
}
```

创建只有一个主分片，没有副本的索引

动态修改副本数：

```json
PUT /my_temp_index/_settings
{
    "number_of_replicas": 1
}
```

##### 为索引配置分析器

我们创建了一个新的分析器，叫做 `es_std` ， 并使用预定义的西班牙语停用词列表：

```json
PUT /spanish_docs
{
    "settings": {
        "analysis": {
            "analyzer": {
                "es_std": {
                    "type":      "standard",
                    "stopwords": "_spanish_"
                }
            }
        }
    }
}
GET /spanish_docs/_analyze?analyzer=es_std
El veloz zorro marrón
结果：
{
  "tokens" : [
    { "token" :    "veloz",   "position" : 2 },
    { "token" :    "zorro",   "position" : 3 },
    { "token" :    "marrón",  "position" : 4 }
  ]
}
```

`es_std` 分析器不是全局的—它仅仅存在于我们定义的 `spanish_docs` 索引中

##### 自定义分析器

什么是一个分析器？组合三种函数，按照顺序执行：**字符过滤器** 整理一个尚未被分词的字符串。例如 html 格式包含 html 标签，可以用其移除，一个分析器可能有0个或多个字符过滤器。**分词器** 一个分析器必须有一个唯一的分词器，把字符串分解成单个词条，并移除掉大部分标点符号。关键词分析器完整输出，空格分词器根据空格切割，正则分析器根据正则表达式匹配 **词单元过滤器** 经过分词，词单元过滤器把单词遏制为词干 lowercase stop 词过滤器

创建一个自定义分析器：

```json
PUT /my_index
{
    "settings": {
        "analysis": {
            "char_filter": { ... custom character filters ... },
            "tokenizer":   { ...    custom tokenizers     ... },
            "filter":      { ...   custom token filters   ... },
            "analyzer":    { ...    custom analyzers      ... }
        }
    }
}
使用 `html清除` 字符过滤器移除HTML部分
自定义的 映射 字符过滤器把 & 替换为 " and "
"char_filter": {
    "&_to_and": {
        "type":       "mapping",
        "mappings": [ "&=> and "]
    }
}
使用标准分词器分词
小写词条，使用小写词过滤器处理
自定义停止词过滤器移除自定义的停止词列表中包含的词
"filter": {
    "my_stopwords": {
        "type":        "stop",
        "stopwords": [ "the", "a" ]
    }
}
"analyzer": {
    "my_analyzer": {
        "type":           "custom",
        "char_filter":  [ "html_strip", "&_to_and" ],
        "tokenizer":      "standard",
        "filter":       [ "lowercase", "my_stopwords" ]
    }
}
汇总：
PUT /my_index
{
    "settings": {
        "analysis": {
            "char_filter": {
                "&_to_and": {
                    "type":       "mapping",
                    "mappings": [ "&=> and "]
            }},
            "filter": {
                "my_stopwords": {
                    "type":       "stop",
                    "stopwords": [ "the", "a" ]
            }},
            "analyzer": {
                "my_analyzer": {
                    "type":         "custom",
                    "char_filter":  [ "html_strip", "&_to_and" ],
                    "tokenizer":    "standard",
                    "filter":       [ "lowercase", "my_stopwords" ]
            }}
}}}
GET /my_index/_analyze?analyzer=my_analyzer
The quick & brown fox
结果
{
  "tokens" : [
      { "token" :   "quick",    "position" : 2 },
      { "token" :   "and",      "position" : 3 },
      { "token" :   "brown",    "position" : 4 },
      { "token" :   "fox",      "position" : 5 }
    ]
}
使用
PUT /my_index/_mapping/my_type
{
    "properties": {
        "title": {
            "type":      "string",
            "analyzer":  "my_analyzer"
        }
    }
}
```

文档字段和属性的三个最重要的设置：

- **`type`**

  字段的数据类型，例如 `string` 或 `date`

- **`index`**

  字段是否应当被当成全文来搜索（ `analyzed` ），或被当成一个准确的值（ `not_analyzed` ），还是完全不可被搜索（ `no` ）

- **`analyzer`**

  确定在索引和搜索时全文字段使用的 `analyzer`

我们将在本书的后续部分讨论其他字段类型，例如 `ip` 、 `geo_point` 和 `geo_shape` 。

