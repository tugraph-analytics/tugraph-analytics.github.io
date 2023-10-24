---
layout: post
read_time: true
show_date: true
show_author: true
title: "开源TuGraph Analytics——是时候开始使用图计算了"
date: 2023-10-23
tags: [图计算, TuGraph-Analytics, 开源, GitHub]
category: opinion
author: TuGraph
description: "开源TuGraph Analytics——是时候开始使用图计算了"
---
作者：TuGraph

## 项目地址： [https://github.com/TuGraph-family/tugraph-analytics](https://github.com/TuGraph-family/tugraph-analytics)
## 项目文档地址: [https://tugraph-analytics.readthedocs.io/en/latest/index_cn/](https://tugraph-analytics.readthedocs.io/en/latest/index_cn/)
## 博客地址： [https://tugraph-analytics.github.io/](https://tugraph-analytics.github.io/)
## 类别：Java
## 项目描述：
GeaFlow(品牌名TuGraph-Analytics)是蚂蚁集团开源的分布式流式图计算引擎，已广泛应用于数仓加速、金融风控、知识图谱以及社交网络等场景。


TuGraph-Analytics的核心能力是以自研图存储为数据底座，支持用户书写SQL+GQL的融合语言，驱动流批一体的流式图计算。
使用TuGraph-Analytics，用户可以继续沿用SQL进行查询，利用引擎的自动转图查询，同时享受图计算极大的性能优势红利。

在任何时候，用户可以无缝切换到GQL等图查询语言，以高效处理复杂多度的关系运算，如3度以上的Join、各类图算法、图特征分析等。
基于图表一体、流批一体的计算引擎，用户可以自然忽略流和批的区隔，只需关注业务需求，计算将在实时计算和离线分析之间无缝切换。

## 使用 TuGraph-Analytics 的好处：
* 支撑万亿级别超大规模图服务的稳定性
* 内建图研发管理平台，一站式图计算
* 业界独创的SQL+GQL融合语言，无需学习，以SQL驱动图计算
* 流批一体的 API 描述，支持自定义图计算任务
* 图和表集成处理
* 基于K8S的云原生部署支持

### 使用案例一 无需学习，零基础开始SQL图计算

TuGraph-Analytics提供融合GQL和SQL样式的查询语言，使得图计算的结果与表查询等价。

这意味着下图中GQL和SQL两种描述都可以达到类似的效果，用户无需预先学习专业图查询语言，即可使用SQL驱动图计算。

![code_style](../../../../assets/images/posts/20230906/code_style.jpg)


GeaFlow DSL引擎层支持了SQL中的Join自动转化为GQL执行，同时支持常用的Project、Filter、Aggregate等一系列SQL操作转为图计算执行。
用户可以自由混用SQL和GQL样式查询，同时做图匹配和表查询。

使用表建模的分析系统只支持SQL join一种方式进行关系分析，这在复杂场景中能力十分局限。
图建模的TuGraph-Analytics支持用户在任何时候，无缝切换到专业性强的图查询语言，以高效处理SQL难以表达的复杂关系运算。

### 使用案例二 进行复杂流图匹配，洞察商业价值
使用专业图计算系统TuGraph-Analytics，只需40行代码，即可搭建一个端到端的循环交易实时检测系统。

首先我们使用历史数据创建交易大图，命名为ethereum_transaction_network。 
接着把来自Kafka的实时交易流table_new_trade不断添加到命名为ethereum_transaction_network的图中。

![image](../../../../assets/images/posts/20230630/kafka_004.png)

每当有新的交易到达的时刻，系统都将触发一次3跳循环交易模式的检查， 把更新的结果存入位于Kafka的外部表tbl_circular_trade，可以很方便地分发给下游组件。

打开一个Kafka Producer，产生消息流，将交易不断发送给Kafka，如左侧终端窗口所示。 最新的循环交易检出结果打印在右侧的Kafka Consumer窗口中。

![image](../../../../assets/images/posts/20230630/kafka_005.png)

当添加一些新的交易日志时，右侧的Kafka Consumer窗口中也实时更新了新的循环交易检出结果，响应十分迅速。

![image](../../../../assets/images/posts/20230630/kafka_006.png)

此案例中，GeaFlow结合Kafka轻松搭建起交易听单->交易网络生成->实时循环交易检出->给下游发送消息完整的金融级实时解决方案。

### 使用案例三 利用内建Console进行研发管理
通过管控平台Console，分析人员可以提交一系列研究作业。 
这些图查询作业会通过GeaFlow引擎自动提交到K8S集群中分布式地运行，大大提高了数据分析的能力和效率。

在GeaFlow Console中新增图任务，任务类型选择“HLA”， 并上传jar包，其中entryClass为算法主函数所在的类。 
点击“提交”，创建任务。
![image](../../../../assets/images/posts/20230815/002.png)

点击”发布”，可进入作业详情界面，点击“提交”即可提交作业。
![image](../../../../assets/images/posts/20230815/004.png)

运行的图查询均可在Console界面查看，方便回溯和管理，并且可在作业详情中查看运行详情，
![image](../../../../assets/images/posts/20230815/005.png)

### 使用案例四 自研图算法，嵌入GQL语言执行
用户可以基于UDGA(User Defined Graph Algorithm)接口实现自定义图算法。
通过将UDGA上传至console平台，可以类似SQL Function的方式在GQL查询中调用。

![image](../../../../assets/images/posts/20230816/tu1.png)

用户可以在GQL图查询语句中嵌入图算法，使用方式如下所示：

```roomsql
INSERT INTO tbl_result
CALL wcc() YIELD (vid, component)
RETURN vid, component
;
```
内置算法或者UDF在BuildInSqlFunctionTable中进行注册。对于非内置算法，可以通过create function语句来创建。

```roomsql
Create funciton wcc as 'com.antgroup.geaflow.dsl.udf.graph.WeakConnectedComponents';
```

## 系统设计图
TuGraph Analytics开源技术架构一共分为五个部分：

* **DSL层**：即语言层。TuGraph Analytics设计了SQL+GQL的融合分析语言，支持对表模型和图模型统一处理。
* **Framework层**：即框架层。TuGraph Analytics设计了面向Graph和Stream的两套API支持流、批、图融合计算，并实现了基于Cycle的统一分布式调度模型。
* **State层**：即存储层。TuGraph Analytics设计了面向Graph和KV的两套API支持表数据和图数据的混合存储，整体采用了Sharing Nothing的设计，并支持将数据持久化到远程存储。
* **Console平台**：TuGraph Analytics提供了一站式图研发平台，实现了图数据的建模、加工、分析能力，并提供了图作业的运维管控支持。
* **执行环境**：TuGraph Analytics可以运行在多种异构执行环境，如K8S、Ray以及本地模式。

![TuGraph Analytics开源技术架构](../../../../assets/images/posts/20230821/tu1.png)

## Benchmark
我们模拟依次执行一跳、两跳和三跳关系运算的场景。足以见得，越是复杂的多跳关系计算，关系模型中Join的性能表现越差。在总时间对比中，利用图的Match计算能够节约超过90%的耗时。
![total_time](../../../../assets/images/posts/20230906/vs_join_total_time_cn.jpg)
<center>图1</center>

------------------------

GeaFlow(品牌名TuGraph-Analytics) 已正式开源，欢迎大家关注！！！

欢迎给我们 Star 哦!

Welcome to give us a Star!

GitHub👉[https://github.com/TuGraph-family/tugraph-analytics](https://github.com/TuGraph-family/tugraph-analytics)

## 微信群
![image](../../../../assets/images/wechat.png)

请点击项目链接下方微信二维码添加微信用户群：[https://github.com/TuGraph-family/tugraph-analytics](https://github.com/TuGraph-family/tugraph-analytics)
