---
layout: post
read_time: true
show_date: true
show_author: true
title: "TuGraph Analytics图建模研发：为图计算业务提速增效"
date: 2023-10-25
tags: [图计算, TuGraph-Analytics, 建模, 提速增效]
category: opinion
author: TuGraph
description: "TuGraph Analytics图建模研发：为图计算业务提速增效"
---
作者：TuGraph

## 概述
GeaFlow Console平台提供了图数据研发能力，包括了对点、边、图、表、函数、任务的管理功能， 为了让用户更好的管理元数据信息，同时也便于用户对图计算进一步地了解。通过对这些研发资源的管理，用户可以方便地、白屏化地创建、修改、删除这些元数据，也可以很方便地查看当前租户下所拥有的数据资产概览及详情，从而更多关注于业务逻辑的实现。
## 图数据研发介绍
### 基本概念

1. 点（GeaflowVertex）：表示一个对象或实体、包含了点id，标签和属性。
2. 边（GeaflowEdge）：表示对象之间的关系，连接点和点，包含源点id，目标点id，标签，时间戳和属性。
3. 图（GeaflowGraph）：表示对象之间关联关系的一种抽象数据结构，由若干个点和若干边组成。
4. 表（GeaflowTable）:  是有结构的数据的集，由行和列组成，每列为字段，有相应的类型和约束条件，每行为一条具体数据。
5. 函数（GeaflowFunction）: 用户自定义的方法，可在dsl作业中使用。
6. 图拓扑（GeaflowEndpoint）: 逻辑概念，为一个三元组<edgeId, srcId, targetId>, 表示在一个图中，一个边的起点和目标点的约束关系。
7. 任务（GeaflowJob）: 研发时对计算逻辑的描述，用户通过编辑任务实现业务逻辑，目前支持dsl和高阶两种方式进行任务研发:
  1. dsl任务（GeaflowCodeJob）: 通过编写dsl代码运行。
  2. 高阶任务（GeaflowApiJob）: 通过api的方式，使用jar包运行。
8. 作业（GeaflowTask）: 由任务经过发布流程生成，最终提交运行。一旦构建，不可修改作业元信息(例如作业的dsl代码或jar包)。
### 模型结构

#### 点&边&图&表&函数
![](../../../../assets/images/posts/20231025/tu0.svg)
Geaflow将所有研发资源进行了结构化的模型设计，从模型图中，可以看到vertex，edge，table都继承自GeaflowStruct，GeaflowStruct中包含一个GeaflowField列表，其中GeaflowField代表了字段，包含类型(Long，String，Boolean等)和约束条件(点id，属性，边源点id，边目标点id等)。在图中，包含了点、边以及Endpoint的列表，描述了实体之间的关联信息。
图和表关联了GeaflowPluginConfig，这是插件配置，表示图表的来源或输出配置，例如odps，sls，oss等。
函数中主要包含一个jar包对象和主类，以标识函数的入口。

#### 任务&作业

![](../../../../assets/images/posts/20231025/tu1.svg)
GeaflowJob为所有任务类型的父类，其中的structs、graphs、functions字段记录了这个任务所使用的图、表、函数，方便用户了解作业关联的一些元信息。任务根据使用途径还可分为以下几种类型:

1. GeaflowProcessJob: 加工任务，用户编写dsl代码实现构图或图计算逻辑, 例如环形计数。
2. GeaflowIntegrationJob: 集成任务，可以根据数据源表自动生成代码，导入到图中。
3. GeaflowCustomJob: 自定义任务， 用户通过编写高阶api代码，实现图计算逻辑，例如实现PageRank, SSSP等算法。
4. GeaflowStatisticJob: 统计任务，用于统计图的信息，例如点边个数。
5. GeaflowServerJob: 查询任务，用于olap服务，执行图查询。

任务（GeaflowJob）和作业（GeaflowTask）通过发布包（GeaflowRelease）进行关联，任务为研发时的描述，作业为运行时的描述，用户可以对任务进行发布，通过BuildPipeline执行流水线构建生成相应的发布包，进而得到相应的作业。Release中包含了作业运行的执行计划、引擎版本、作业参数、集群和集群参数等，是作业在运行时所需要的信息。用户通过创建和修改任务进行业务逻辑的研发，通过发布的作业进行提交运行。

## Demo演示

### 创建点
在研发管理中新增点定义, 每个点有对应的字段列表，且必须有点id字段，如下例子中新增了2个点：person和software。
![](../../../../assets/images/posts/20231025/tu2.png)


![](../../../../assets/images/posts/20231025/tu3.png)

### 创建边
在研发管理中新增边定义, 每条边需要有源点id和目标点id字段，如下例子中新增了2条边：knows和creates。
![](../../../../assets/images/posts/20231025/tu4.png)

### 创建图
在研发管理中新增图定义, 图可以关联之前定义的点和边，console中通过选择框的方式进行关联。如下例子中，创建了名为dy_modern的图，其包含了person和software点、created和knows边。同时，可以为图配置拓扑约束，限制此图上边的源点目标点的绑定关系，例如create边只能是person->software, know边只能是person->person（Endpoint具体作用将在后续文章中介绍）。
![](../../../../assets/images/posts/20231025/tu6.png)

### 创建表
在研发管理中新增表定义， 此例子中创建了一个输出表，为最终结果输出的载体，有2个字段person名字和software名字。其参数配置中的类型为file，表示输出到本地文件目录(也可以选择其他类型，例如kafka，hive)。
![](../../../../assets/images/posts/20231025/tu7.png)


![](../../../../assets/images/posts/20231025/tu8.png)

### 创建任务
本示例中，构造如下关系图:
![](../../../../assets/images/posts/20231025/tu9.jpg)

任务dsl如下，先向图dy_modern中插入点边数据，然后执行MATCH遍历图，找到id=1的人（jim）认识的人（kate、tom）所创建的软件（software1、software2），最后将结果插入到tbl_result表（文件）中。
```sql
USE GRAPH dy_modern;

INSERT INTO dy_modern.person(id, name, age)
SELECT 1, 'jim', 20
UNION ALL
SELECT 2, 'kate', 22
UNION ALL
SELECT 3, 'tom', 24
;

INSERT INTO dy_modern.software(id, name, lang)
SELECT 4, 'software1', 'java'
UNION ALL
SELECT 5, 'software2', 'java'
;

INSERT INTO dy_modern.knows
SELECT 1,2 ,0.2
UNION ALL
SELECT 1,3 ,0.3
;

INSERT INTO dy_modern.creates
SELECT 2, 4, 0.6
UNION ALL
SELECT 3, 5, 0.8
;

INSERT INTO tbl_result
SELECT
	b_name,
    c_name
FROM (
  MATCH (a:person where id = 1) -[e:knows]->(b:person)-[e2:creates]-> (c:software)
  RETURN b.name as b_name, c.name as c_name
)
```
![](../../../../assets/images/posts/20231025/tu10.png)

任务发布之后即可生成对应task，进入作业详情界面，提交作业之后开始执行，最终运行完成。
![](../../../../assets/images/posts/20231025/tu11.png)

作业的参数配置如下, **注意worker数需要设置**:
![](../../../../assets/images/posts/20231025/tu12.png)


![](../../../../assets/images/posts/20231025/tu13.png)

在容器的/tmp/result目录中找到结果文件。
```console
[root@09db8348371a tmp]# cd /tmp/result/
.partiton_0.crc  .partiton_1.crc  partiton_0  partiton_1
```
由于graph中的shradCount设置的为2，所以结果文件有2个分片：partition_0, partition_1
查看文件，有2条数据，符合结果。

```txt
kate,software1
tom, software2
```


**至此，我们就成功使用平台的图研发功能完成了图表的创建和计算作业的运行！是不是超简单！快来试一试吧！**

------------------------

GeaFlow(品牌名TuGraph-Analytics) 已正式开源，欢迎大家关注！！！

欢迎给我们 Star 哦!

Welcome to give us a Star!

GitHub👉[https://github.com/TuGraph-family/tugraph-analytics](https://github.com/TuGraph-family/tugraph-analytics)

## 微信群
![image](../../../../assets/images/wechat.png)

请点击项目链接下方微信二维码添加微信用户群：[https://github.com/TuGraph-family/tugraph-analytics](https://github.com/TuGraph-family/tugraph-analytics)
