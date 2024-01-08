---
layout: post
read_time: true
show_date: true
show_author: true
title: "TuGraph Analytics交互式图查询：让图所见即所得"
date: 2023-12-13
tags: [图计算, 开源, 交互查询, 图分析, 蚂蚁集团, OLAP]
category: news
author: 廖梵抒
description: "TuGraph Analytics交互式图查询：让图所见即所得"
---
作者：廖梵抒

TuGraph Analytics提供了OLAP图分析能力，实现图上的交互式查询，用户在构图并导入数据之后，可以通过输入GQL语句对图查询分析，并以可视化的方式直观地展示点边结果。

# OLAP架构

![OLAP架构](https://picx.zhimg.com/80/v2-6e19f3b416b88b1fb0c572e30b51325f_1440w.png)

在TuGraph Analytics OLAP架构中，主要以下组件:

1. **Client**: 用户通过Client提交查询语句,  Client负责和Coordinator交互，发送查询请求。
2. **Coordinator**: 接收来自Client查询请求，将查询中的GQL语句进行解析、优化，构建查询的执行计划（执行计划的生成逻辑可参考[《分布式图计算如何实现？带你一窥图计算执行计划》](https://zhuanlan.zhihu.com/p/647441899)），并将任务调度给Woker执行。
3. **Worker**：具体分布式地执行任务的单元，接收到Coordinator发送的Pipeline，执行具体的计算和查询逻辑。
4. **Meta Service**: 服务注册管理，Coordinator启动后，会将服务的地址和端口向MetaService进行注册，Client提交查询时从MetaService获取Coordinator的服务地址，进行连接。目前支持http和rpc两种方式。

组件间执行流程如下：
![OLAP流程](https://pic1.zhimg.com/80/v2-05bcbbdc7d37580514bdf1244ec31948_1440w.png)

# 操作指南

## 1. 定义图模型

以下图为例，图中有2种点person和software，以及2种边knows和creates。

![图模型](https://pic1.zhimg.com/80/v2-8373a1ac176d7a78af87e2a6341dcaf6_1440w.jpg)

图模型定义可参考[《TuGraph Analytics图建模研发：为图计算业务提速增效》](https://zhuanlan.zhihu.com/p/663270153)，图定义语法为：

```sql
CREATE GRAPH dy_modern (
	Vertex person (
	  id bigint ID,
	  name varchar,
	  age int
	),
	Vertex software (
	  id bigint ID,
	  name varchar,
	  lang varchar
	),
	Edge knows (
	  srcId bigint SOURCE ID,
	  targetId bigint DESTINATION ID,
	  weight int
	),
	Edge creates (
	  srcId bigint SOURCE ID,
  	targetId bigint DESTINATION ID,
  	weight int
	)
) WITH (
	storeType='rocksdb',
	shardCount = 2
);
```

## 2. 准备图数据

创建“加工”类型图任务，发布生成图作业。

```sql
USE GRAPH dy_modern;

INSERT INTO dy_modern.person(id, name, age)
SELECT 1, 'jim', 20
UNION ALL
SELECT 2, 'kate', 22
UNION ALL
SELECT 3, 'tom', 24;

INSERT INTO dy_modern.software(id, name, lang)
SELECT 4, 'software1', 'java'
UNION ALL
SELECT 5, 'software2', 'java';

INSERT INTO dy_modern.knows
SELECT 1,2,2
UNION ALL
SELECT 1,3,3
UNION ALL
SELECT 3,2,3;

INSERT INTO dy_modern.creates
SELECT 2,4,6
UNION ALL
SELECT 3,5,8
UNION ALL
SELECT 3,4,8;
```

**图作业需要的worker数为23**，在作业界面将参数进行修改，之后提交作业运行。

![Worker数配置](https://picx.zhimg.com/80/v2-84ab98bce69df987ced761d3b779aa8b_1440w.png)

## 3. 创建查询服务

创建图查询服务, 任务类型选择“图查询”，目标图选择刚才创建的图。

![创建查询服务](https://picx.zhimg.com/80/v2-52464d10623b2a0cd178307c318fc777_1440w.png)

发布任务后，使用默认参数即可，提交作业。

## 4. 执行查询

图查询服务的作业变成RUNNING状态后，可在任务界面点击“查询”进入图查询界面

![进入图查询](https://picx.zhimg.com/80/v2-99522f7218d76bb69c5376b3d1840e91_1440w.png)

输入相应的gql查询语句，点击“执行”，即可得到查询结果。

![执行图查询](https://pic1.zhimg.com/80/v2-3ead72433f3e97b40e812248f32f65a5_1440w.png)

## 5. 图可视化

点击某个点，可以查看点关联的具体信息和属性，以及关联的其他点边。

![点边视图](https://pic1.zhimg.com/80/v2-3ba5b99d91bc50700503f2b90ce370b7_1440w.png)

除了可视化的方式，也可以json形式看到返回的结果。

![JSON视图](https://picx.zhimg.com/80/v2-80591a04c52dbb29c47094981420c402_1440w.png)

**至此，我们就成功使用TuGraph Analytics实现了图上的交互式查询！是不是超简单！快来试一试吧！**

**欢迎关注我们的GitHub仓库：** 👉 [**https://github.com/TuGraph-family/tugraph-analytics**](https://github.com/TuGraph-family/tugraph-analytics)

------------------------

GeaFlow(品牌名TuGraph-Analytics) 已正式开源，欢迎大家关注！！！

欢迎给我们 Star 哦!

Welcome to give us a Star!

GitHub👉[https://github.com/TuGraph-family/tugraph-analytics](https://github.com/TuGraph-family/tugraph-analytics)

## 微信群
![image](../../../../assets/images/wechat.png)

请点击项目链接下方微信二维码添加微信用户群：[https://github.com/TuGraph-family/tugraph-analytics](https://github.com/TuGraph-family/tugraph-analytics)
