---
title: 常用 API 介绍
weight: 5
chapter: false
draft: false
---

前面介绍了 Elasticsearch 和 Kibana 的安装，这一节主要来介绍一下在 Kibana 里面的 DevTools 下，具体如何来进行最基本的 Elasticsearch 操作。Elasticsearch 大部分 API 的输入输出都是 JSON 数据结构。有关 JSON 的更完整的介绍可访问：
[https://www.json.org/json-zh.html](https://www.json.org/json-zh.html)

## 增删改查

首先是索引的操作，其实在第一节关于 Elasticsearch 安装的时候，已经介绍过，我们再来回顾一下。

假设我们创建一个有包含 2 个字段的索引文档。

```js
POST twitter/_doc/1
{
  "name":"jack",
  "age":30
}
```

返回结果为：

```js
{
  "_index": "twitter",
  "_type": "doc",
  "_id": "1",
  "_version": 1,
  "result": "created",
  "_shards": {
    "total": 2,
    "successful": 1,
    "failed": 0
  },
  "_seq_no": 0,
  "_primary_term": 1
}
```

如果要取回这个文档，只需要将 POST 换成 GET，如下：

```js
GET twitter/_doc/1
```

返回结果为：

```js
{
  "_index": "twitter",
  "_type": "doc",
  "_id": "1",
  "_version": 1,
  "found": true,
  "_source": {
    "name": "jack",
    "age": 30
  }
}
```

返回结果里面的 `found` 说明找到了该文档，`_source` 即我们 POST 进去的原始文档。

如果要修改这个文档，有两种方式，一种是完全替换，使用相同的路径即可，我们修改 age 为 35，如下：

```js
PUT twitter/_doc/1
{
  "name":"jack",
  "age":35
}
```

返回结果：

```js
{
  "_index": "twitter",
  "_type": "doc",
  "_id": "1",
  "_version": 2,
  "result": "updated",
  "_shards": {
    "total": 2,
    "successful": 1,
    "failed": 0
  },
  "_seq_no": 1,
  "_primary_term": 1
}
```

还一种更新叫做部分更新，你只需指定要更新的字段和该字段的值即可，不用准备完整的 JSON 文档，如下：

```js
POST twitter/_update/1
{
    "doc" : {
        "name" : "mark"
    }
}
```

最外层的 `doc` 是固定的结构，我们只需要准备要更新的字段放在里面就行。

最后我们再来看看删除，只需要把取回文档操作的 GET 换成 DELETE 就行了，如下：

```js
DELETE twitter/_doc/1
```

返回：

```js
{
  "_index": "twitter",
  "_type": "doc",
  "_id": "1",
  "_version": 4,
  "result": "deleted",
  "_shards": {
    "total": 2,
    "successful": 1,
    "failed": 0
  },
  "_seq_no": 3,
  "_primary_term": 1
}
```

很好，`result` 显示删除成功。


## 搜索的使用

我们创建两个索引文档，ID 分别是 1 和 2，如下：

```js
POST twitter/_doc/1
{
  "name":"jack",
  "age":30
}

POST twitter/_doc/2
{
  "name":"mark",
  "age":35
}
```

现在我们想通过名称来进行搜索，可以使用 `_search` 接口，然后指定条件 `q` 来传递查询关键字，如下：

```js
GET twitter/_search?q=jack
```

返回：

```js
{
  "took": 51,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": 1,
    "max_score": 0.2876821,
    "hits": [
      {
        "_index": "twitter",
        "_type": "doc",
        "_id": "1",
        "_score": 0.2876821,
        "_source": {
          "name": "jack",
          "age": 30
        }
      }
    ]
  }
}
```

`hits` 就是我们搜索返回的结果，`total` 是搜索命中的结果总数，下面的 `hits` 是本次返回的部分搜索结果，是一个数组结构，里面是具体的索引文档。

我们再看看如何通过年龄来进行检索，如下：

```js
GET twitter/_search?q=35
```

返回：

```js
{
  "took": 9,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": 1,
    "max_score": 1,
    "hits": [
      {
        "_index": "twitter",
        "_type": "doc",
        "_id": "2",
        "_score": 1,
        "_source": {
          "name": "mark",
          "age": 35
        }
      }
    ]
  }
}
```

另外，我们还能限定查询的字段，如只对年龄字段进行查询，如下：

```js
GET twitter/_search?q=age:35
```

除此以外，还能使用 QueryDSL 来写查询表达式，即将查询表达式以 JSON 的方式来通过请求体（request body）来传递，而不是走 URL 网址参数，如下：

```js
GET twitter/_search
{
  "query": {
    "match": {
      "age": 35
    }
  }
}
```
搜索结果应该和前面 url 传参效果一致，读者可以自行尝试一下。

## 聚合的使用

聚合是 Elasticsearch 里面用来对搜索结果的数据进行统计和聚合操作的过程，通过聚合，我们可以对搜索结果做到按各个维度的统计，结合丰富的前端展现，从而实现良好的搜索体验。

我们再创建两个索引文档，如下：

```js
POST twitter/_doc/3
{
  "name":"john",
  "age":30
}
POST twitter/_doc/4
{
  "name":"mark",
  "age":40
}
```

现在我们想统计一下，年龄的分布情况，可以使用如下的查询条件，最外层的 `size` 为 `0` 这样可以不返回搜索命中的文档，只返回聚合的统计结果，`aggs` 就是我们描述聚合查询语句根节点，我们使用 `terms` 聚合类型来对 `age` 字段的值进行统计，并且只返回前 `10` 个统计值，然后这些统计结果我们命名为 `age_stats`，如下图：

```js
GET twitter/_search
{
  "size": 0,
  "aggs": {
    "age_stats": {
      "terms": {
        "field": "age",
        "size": 10
      }
    }
  }
}
```

返回：

```js
{
  "took": 55,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": 4,
    "max_score": 0,
    "hits": []
  },
  "aggregations": {
    "age_stats": {
      "doc_count_error_upper_bound": 0,
      "sum_other_doc_count": 0,
      "buckets": [
        {
          "key": 30,
          "doc_count": 2
        },
        {
          "key": 35,
          "doc_count": 1
        },
        {
          "key": 40,
          "doc_count": 1
        }
      ]
    }
  }
}
```

可以看到查询返回的 JSON 里面包含了表示聚合结果的 `aggregations` 节点，下面有我们的统计结果 `age_stats`，下面 `buckets` 表示聚合的具体值 `key` 和统计数据 `doc_count`，也就是各个年龄分别有多少个文档，出现了多少次。

## 索引的管理

Elasticsearch 还有很多 API 可以用来对索引进行管理，比如我们还可以通过 `_cat` API 来查看索引列表:

```js
GET _cat/indices?v
```

返回：

```js
health status index   uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   .kibana 3UiUi8MZQDeF3p97eRWyFw   1   0          2            1      9.7kb          9.7kb
yellow open   index   VpNv75DzQESBfX8DcLLcww   5   1          1            0      4.3kb          4.3kb
```

返回的结果不是 JSON，而是精简的表格类型，第一行是字段名，后面行是具体的数据，可以看到索引的列表、各索引状态、文档数和索引大小等参数，各字段分别说明如下：

| 字段名 | 说明 |
| --- | --- |
| health |  索引健康状态，`green` 表示健康；`yellow` 表示数据完整，但是缺少副本；`red` 则表示有数据损坏。 |
| status |  索引工作状态，`open` 表示索引打开中，可以被使用；`close` 表示索引被关闭，不能使用。 |
| index |  索引名称。 |
| uuid |  索引的唯一 ID 标识。 |
| pri |  索引的主分片个数。 |
| rep |  索引的副本分片个数。 |
| docs.count |  索引内的文档个数。 |
| docs.deleted |  索引内已经删除的文档个数。 |


如果要删除索引，可以执行下面的动作来清理整个索引，`twitter` 就是其中要删除的索引名：

```js
DELETE twitter
```

集群管理的接口很多，我们这里不展开，我们只需要能够知道查看索引是否存在、索引有多少文档，索引是否正常和能够删除索引，我们就可以继续后面的实战了。

## 集群的监控

除此之外，我们还可以查看集群的健康状况：

```js
GET _cluster/health
```

返回：

```js
{
  "cluster_name": "elasticsearch",
  "status": "yellow",
  "timed_out": false,
  "number_of_nodes": 1,
  "number_of_data_nodes": 1,
  "active_primary_shards": 14,
  "active_shards": 14,
  "relocating_shards": 0,
  "initializing_shards": 0,
  "unassigned_shards": 13,
  "delayed_unassigned_shards": 0,
  "number_of_pending_tasks": 0,
  "number_of_in_flight_fetch": 0,
  "task_max_waiting_in_queue_millis": 0,
  "active_shards_percent_as_number": 51.85185185185185
}
```

这里我们只需要关注 `status` 字段，当值为 `green` 或者 `yellow` 时，我们就可以正常使用该 Elasticsearch 服务。
