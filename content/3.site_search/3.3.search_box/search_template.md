---
title: 搜索模板
weight: 11
chapter: false
draft: false
---

本节我们主要讨论怎么优化查询的构造方式，前面章节提到我们的一个 QueryDSL 是一个复杂的 JSON，我们每次在构造查询的时候，都需要进行各种参数的处理，然后再拼接出最终的 QueryDSL，首先是处理逻辑会变得很复杂，需要处理每种参数赋值与否的各种场景，最头疼的是，每次加参数的时候，都需要重新修改拼装逻辑，实在是头痛，那么有没有一种简洁优雅的方法来处理查询的构建呢。

答案自然是有的，那就是使用搜索模板（search_template），具体怎么使用呢？我们一起来看一下。

## 将查询和参数分离

Elasticsearch 里面的 [Search Template](https://www.elastic.co/guide/en/elasticsearch/reference/6.6/search-template.html) 允许我们提前定义好一个查询的模板，并且通过参数的方式来接受外部传进来的变量，然后借助 Elasticsearch 内置的 Mustache 脚本引擎来进行渲染，从而得到最终的 QueryDSL，借助搜索模板，我们从而进行查询条件和具体查询语句的分离解耦。

先来看一个简单的通过搜索模板来执行查询的例子，如下图：

```
GET _search/template
{
    "source" : {
      "query": { "match" : { "{{my_field}}" : "{{my_value}}" } },
      "size" : "{{my_size}}"
    },
    "params" : {
        "my_field" : "name",
        "my_value" : "张三",
        "my_size" : 10
    }
}
```

这个 JSON 请求体主要包含两个参数，一个是 source 用来定义我们的模板，里面可以看到 query、size 都是我们常见的查询参数，比如这里使用的是 match 查询，字段参数是 my_field，`{{}}` 是 mustache 的语法，表示这个是引用的一个变量名，那么变量是怎么来的呢，我们看到下面有一个 params 节点，下面就是定义了这些变量的值。我们这个模板最后渲染的查询条件为：

```
{
      "query": { "match" : { "name" : "张三" } },
      "size" : "10"
 }

```

我们通过执行 `_search/template` 这个 API 就能够动态传参并渲染查询条件从而得到查询结果。
不过如果还是需要传完整的这样一个模板定义，那么还是没有达到我们的目的啊，好在 Elasticsearch 早想到了，我们可以把上面的模板单独存在一个地方，在使用的时候，通过模板 ID 引用就可以了。

### 小知识
- 调试模板渲染结果： `GET _render/template`
- 取回模板定义的语法： `GET _scripts/<templatename>`
- 删除模板定义的语法： `DELETE _scripts/<templatename>`
- 模板变量的输出：`{{query_string}}`
- 判断模板变量：`{{#line_no}}中间为当变量存在会输出的代码块{{/line_no}}`

## 功能集成
还是延续社区文章数据的例子，我们先创建一个搜索模板对象，模板 ID 我们加上版本号，为了方便以后支持不同的版本，我们将该搜索模板取名为 `forum_search_template_v1` ，如下：

```
POST _scripts/forum_search_template_v1
{
  "script": {
    "lang": "mustache",
    "source": {
      "size": "{{size}}",
      "query": {
        "match": {
          "{{field}}": "{{query}}"
        }
      },
      "_source": [
        "title",
        "id",
        "uid",
        "views"
      ]
    }
  }
}
```

现在我们看看如何来调用这个模板，并动态的传参，示例如下：

```
GET forum-mysql/_search/template
{
  "id": "forum_search_template_v1",
  "params": {
    "field": "title",
    "query": "elasticsearch",
    "size": 10
  }
}
```
### 小知识
- 通过设置一个或者多个索引来应用索引模板的方法：`GET index1,index2*/_search/template`

上面的这个查询看起来是不是很清爽，执行上面的语句就能得到我们的查询结果了，更重要的是我们实现了查询条件和参数的分离。

接下来，我们只需要修改我们的 PHP 代码，替换之前硬编码的 QueryDSL 查询为我们新的模板调用就好了，如下图：

![](/media/15486660133014/15490315444263.jpg)


由于我们的 URL 是来自于后台的动态设置，我们同样需要进行相应的修改就可以了，如下图：

![](/media/15486660133014/15490182676340.jpg)

就算以后我们需要调整查询条件，我们只需要动态的修改搜索模板就好了，网站代码完全不用调整和重新部署，实在是非常的方便啊。


### 小知识
- 设置模板参数的默认值 `{{var}}{{^var}}default{{/var}}`
- 相关变量定义因为包含大括号 `{{var}}` 在source里面必须是包含在双引号内，否则请求体会变成不合法的 JSON，或者可以对整个 `source` 进行转义。
- 将数组或者复杂对象直接转换为 JSON 对象 `{{#toJson}}parameter{{/toJson}}`
- 将数组拼接成字符拼接的字符串 `{{#join delimiter='||'}}date.formats{{/join delimiter='||'}}`
