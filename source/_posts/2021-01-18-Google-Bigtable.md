---
title: Google Bigtable
date: 2021-01-18 16:58:40
categories:
tags:
---
## 1.介绍
在2003年底，我们设计，实现并部署了在Google内部名为Bigtable的分布式存储系统，Bigtable能够管理结构化的数据。Bigtable被设计为在PB级别数据处理和数以百计的机器上能够具有可靠的伸缩性。Bigtable实现了几个目标：广泛的适用性，伸缩性，高性能和高可用。Bigtable已经被应用于6个Google 产品和项目中，包括Google Analytics, Google Finance, Orkut, Personalized Search, Writely, and Google Earth。这些产品使用Bigtable处理各种苛刻的工作负载，从面向吞吐量的批处理作业到面向用户的延迟敏感服务。Bigtable集群被这些产品用于跨越从少数到多数的服务器的各种配置，存储高达数百TB。

在很多方面，Bigtable表现得很像数据库：它提供了很多的实现策略像数据库一样。并行数据库和主存数据库已经实现了可伸缩性和高可用，但是Bigtable提供了与这些系统不同的接口。Bigtable不支持全关系型数据模型；作为代替的，它提供了客户端一种简单的数据模型，支持对数据布局和格式的动态控制，并允许客户端对底层存储中表示数据的局部属性（译者注：类似元数据）进行推理。数据能被任意字符串的行和列名进行索引。Bigtable还将数据视为未解释的字符串，尽管客户端经常将各种形式的结构化和半结构化数据序列化到这些字符串中。客户端能够在schemas进行细粒度的控制数据的位置。Bigtable schema参数允许客户端动态的控制是从内存还是从磁盘提供数据。

第2节更为详细的描述了数据模型，第3节概述了客户端API。第4节简要的描述了Bigtable所依赖的Google底层基础设施。第5节描述了Bigtable实现的基本原理，第6节描述了我们为了提高Bigtable性能所做的更新。第7节提供了Bigtable性能的度量。我们再第8节中描述了Bigtable在Google中实现的几个例子，并在第9节讨论一些我们在设计和支持Bigtable中学到的一些教训。最后，第10节描述了相关工作，第11节给出了我们的总结。

## 2.数据模型
Bigtable集群是运行Bigtable软件的一组处理进程。每个集群服务一组表。Bigtable的表是稀疏的，分布式的，持久化的多维排序map。数据被划分为3个维度：行，列，时间戳。

(row:string, column:string, time:int64) → string

我们将特定的行键，列键和时间戳引用的存储称为一个单元（译者注：类似关系数据库的元数据组定义一个数据）。

![图1  存储网页的示例表的一部分。行名将URL进行了反转。contents列家族包含页面内容，超链接列家族包含了引用该页面的文本。CNN的主页被《体育画报》和“我的外观”主页所引用，所以改行包含了列名为anchor:cnnsi.com 和 anchor:my.look.ca。每个超链接单元格有1个版本；contens列有3个版本，时间戳分别是t3，t5，t6。](https://cdn.jsdelivr.net/gh/fuxyzz/cdn/images/bigtable_1.png)
