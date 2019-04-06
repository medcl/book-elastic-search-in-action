---
title: 数据结构
weight: 8
chapter: false
draft: false
---

首先交代一下本机 MySQL 的基本信息：


| 属性  | 值 |
| :-- | :-- |
| host  | 127.0.0.1 |
| port | 3306 |
| username | root |
| password | qeephp |
| dbname | qa_es |
| charset | utf8mb4 |


社区网站的数据里面有很多张表，我们先只需要关心文章相关的表即可，表名
`aws_article`，我们执行一条 SELECT 查询语句，如下图：

![](/media/15287928224682/15287973833321.jpg)

很好，还看到数据了，我们暂时只需要关心 `title` 和 `message` 字段就好了，因为我们第一个搜索任务主要搜索这两个字段就可以。

然后我们回忆起基础准备里面介绍的，Elasticsearch 索引的是 JSON 格式的文档，我们参照数据库字段，定义一个 JSON 文档，如下：

```js
{
"title":"hello world",
"message":"i am hello world"
}
```

是不是就行了？NO，我们试想一下，搜索得到这个结果之后，是不是还需要打开对应的网页啊，那要把唯一标识也加上才能生成对应的详情页网址进行跳转，另外显示结果的时候，是不是还要显示一下作者的信息呢，是不是还要显示一下发布时间呢，是不是还要显示评论个数和查看次数呢，是不是还要显示分类信息呢，统统有必要！这些信息越完整，用户越能尽早查看并做决策是否要进行点击来查看详情页面，所以，我们也要把这些字段加上。

我们把 Mock 的这个 JSON 示例索引文档提交到索引 `forum` 里面：

```js
POST forum/doc/1
{
  "title": "hello world",
  "message": "i am a simple message",
  "id": "62",
  "uid": "1038",
  "comments": "1",
  "views": "1231",
  "addtime": "1456042765",
  "votes": "12",
  "category_id": "2"
}
```

并执行一个查询，如下图：

```js
GET forum/_search
{
  "query": {
    "multi_match": {
      "query": "i am a simple message"
    }
  }
}
```

查询结果：

```js
{
  "took": 0,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": 1,
    "max_score": 0.5753642,
    "hits": [
      {
        "_index": "forum",
        "_type": "doc",
        "_id": "1",
        "_score": 0.5753642,
        "_source": {
          "title": "hello world",
          "message": "i am a simple message",
          "id": "62",
          "uid": "1038",
          "comments": "1",
          "views": "1231",
          "addtime": "1456042765",
          "votes": "12",
          "category_id": "2"
        }
      }
    ]
  }
}
```
可以看到，我们已经能够通过关键字搜索到这个文档，虽然我们数据库的数据还没真的进入到 Elasticsearch 里面，此处也不是真实数据，但是我们就可以提前在Elasticsearch 里面进行搜索功能验证。这个也是 Elasticsearch 的强大之处。非常简单且可大大提高开发效率和减少原型验证时间。

下一步，我们只要想办法把数据库里面的行列格式的数据转换成上面的 JSON 文档格式就可以了。
