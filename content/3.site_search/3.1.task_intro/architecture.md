---
title: 总体架构
weight: 7
chapter: false
draft: false
---

确定好了功能范围之后，我们再来分析一下目前的网站的架构和新的架构。

现有架构， 假定版本为 v1.0：

PHP + MySQL
【无图】

没错，就是这么简单，PHP 直接访问 MySQL 数据库，没有各种高大上的组件，我都不用画图了！

>注：PHP 和 MySQL 的知识储备就不在本书讨论了，大家自行搜索解决吧。

新的架构，引入了 Elasticsearch 来提供更好的搜索体验，改进版本即为 v1.1：

PHP + MySQL + Elasticsearch
【无图】

同样非常简单！

本机的 PHP 和 MySQL的版本分别是：

- PHP Version 5.4.25
- MySQL Version 5.5.34
