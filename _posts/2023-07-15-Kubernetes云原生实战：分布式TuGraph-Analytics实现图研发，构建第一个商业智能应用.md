---
layout: post
read_time: true
show_date: true
title: "Kubernetes云原生实战：分布式GeaFlow实现图研发，构建第一个商业智能应用"
date: 2023-07-04
tags: [K8S, 云原生, 图研发, 商业智能, GeaFlow, BI, 数据分析]
category: opinion
author: 林力韬
description: "Kubernetes在云原生应用中扮演着至关重要的角色，为商业智能（BI）强大赋能。不同于传统的BI，容器化部署在集群中可以获得更高的可靠性、弹性和灵活性。"
---
author: 林力韬 

## 引言
Kubernetes在云原生应用中扮演着至关重要的角色，为商业智能（BI）强大赋能。 不同于传统的BI，容器化部署在集群中可以获得更高的可靠性、弹性和灵活性。

但在实际生产实践中，这还远远不够。 商业智能分析人员更希望搭建实时提问快速响应的研发平台，使得数据能够回应分析人员的想法，并产出更多支持商业决策的信息。 这需要类似于OLAP的在线分析处理（OLAP）技术，帮助查询、分析和理解大规模数据，从而做出更明智的商业决策。

例如这样一个接近实际的场景——

分析师想要找到具有'Comedian'类别下标签的博文和评论，TA的意图可以被描述如下:
```sql
MATCH (:TagClass where name = 'Comedian')
      <-[:hasType]-(:Tag)
      <-[:hasTag]-(msg:Post|Comment)
```
这里使用了开源图研发引擎GeaFlow支持的GQL语言描述，可以简单直观地描述商业关系。 后文将介绍基于分布式GeaFlow实现图研发，都采用类似的描述。

一段时间的研究后，分析师开始关注发布者的朋友都有哪些：
```sql
MATCH (:Tag)
      <-[:hasTag]-(msg:Post)
      <-[:hasCreator]-(person:Person)
      -[:knows]-{0,1}(friend:Person)
```

熟悉关系型数据库和大数据处理平台的朋友可能知道，这类模式分析涉及一系列的大型Join连接处理，在现有系统中计算时间往往超出预期。 在传统的商业智能模式下，分析师可能需要等待数天的时间才能获得他们需要的结果，以上3个小查询的处理成本已经大大超出它们产出的价值。

因此，我们需要一种能够快速支持查询和业务演进的解决方案，并且能够在一次构建后持续不断地提供商业智能信息产出。 在这种情况下，最好的选择是使用云原生部署的图研发平台。

图研发平台可以提供高效的图分析能力，使得分析师可以更快地探索数据，并且可以轻松地构建和优化查询。 使用图研发平台，可以将复杂的数据模型转化为可视化的图形表示，使得即使是新手分析师也可以直观地理解数据之间的关系。 此外，云原生技术保证平台的可扩展性和弹性，使得一行代码可以瞬间放大几十万倍，支撑起大规模数据的快速分析。

接下来我们带着这个问题出发，以支持云原生的分布式图研发平台GeaFlow为例，快速搭建起你的第一个商业智能应用。

## 部署环境

部署GeaFlow需要一个docker+K8S的云原生环境，因此需要提前安装docker和K8S。 GeaFlow能够将业务数据转化为图，一旦将数据导入一张图中，后续就可以持续支持各种分析需求。 业务数据支持各类来源，包括数据库、Hive、Kafka等等。

GeaFlow会在镜像中自动拉起MySQL、Redis、RocksDB、InfluxDB等必须组件。

### 部署K8S

GeaFlow依赖K8S运行图研发作业，安装K8S后需要取得API地址。

K8S API Server默认监听6443端口，可以在K8S集群的任一节点上使用以下命令查看集群信息：
```sh
kubectl cluster-info
```
查看输出结果中的“Kubernetes master”字段，该字段将列出Master节点的URL地址，该地址即为K8S API地址，例如：
```console
Kubernetes master is running at https://172.25.8.152:6443
```

'/etc/kubernetes/admin.conf' 是Kubernetes集群的管理员配置文件，其中包含了客户端程序访问Kubernetes API Server的必要信息。

* kubernetes.ca.data：该字段包含一个Base64编码的PEM格式的CA证书，用于验证Kubernetes API Server的身份。
* kubernetes.cert.data：该字段包含一个Base64编码的PEM格式的客户端证书，用于验证客户端的身份。
* kubernetes.cert.key：该字段包含一个Base64编码的PEM格式的私钥，用于解密客户端证书。

这些字段的值通常被编码为Base64格式并存储在配置文件中，以保证安全性。

GeaFlow需要取得这三个参数以启动K8S客户端，提交作业到集群。

### 部署DFS
为了存储TB级别的超大规模图，可能需要搭建DFS，小规模数据则不必要部署Hadoop，GeaFlow可以将图数据保存在本地磁盘中。

部署Hadoop后，需要取得文件系统地址，GeaFlow需要连接Hadoop写入图数据和系统运行状态数据。 打开终端并登录到Hadoop集群中的任何一个节点，运行以下命令：

```sh
hdfs getconf -namenodes
```

这将显示Hadoop集群的所有主节点的列表。如果集群只有一个主节点，则只会显示一个主节点的名称。 连接到Hadoop集群的主节点，运行以下命令：

```sh
hadoop org.apache.hadoop.conf.Configuration | grep 'fs.defaultFS'
```

这将显示fs.defaultFS的值，即Hadoop集群的默认文件系统URI。

### 部署外部数据源
GeaFlow目前支持DFS、Kafka、Hive等数据源，未来将支持JDBC、Pulsar等数据源。

用户可以实现自定义的数据源，参考[自定义Connector文档](https://tugraph-analytics.readthedocs.io/en/latest/docs-cn/application-development/dsl/connector/udc/)。其中也包含现有数据源的使用方法。

如果需要更多数据源的支持，可以通过GitHub项目地址提出ISSUE，或者加入微信群联系我们。

## 安装GeaFlow

GeaFlow提供一个分布式图计算引擎GeaFlow，同时提供一个完整的图研发管控平台Console。 用户可在系统内完成图数据创建、研发、运维等工作。 管控平台Console可以基于Docker独立启动，配置好集群和存储系统后，图研发作业可以方便地提交到K8S集群运行。

参考[GeaFlow安装部署文档](https://tugraph-analytics.readthedocs.io/en/latest/docs-cn/deploy/install_guide/)安装GeaFlow。

在集群配置步骤中，配置K8S集群到GeaFlow，填入K8S服务地址与前文提到的证书信息。

![image](../../../../assets/images/posts/20230715/anzhuang1.png)


安装时会提示**当前为单机部署模式**，这表示Console平台使用默认单机部署。 图研发作业会被提交到配置的K8S集群，Console平台提供作业编辑和运维能力，不受影响。

在数据存储配置步骤中，配置导入GeaFlow的图数据存储位置。

![image](../../../../assets/images/posts/20230715/anzhuang3.png)

数据量较小可以配置为LOCAL模式，无需修改。 若数据量比较大，配置为DFS地址，Root路径为存储数据在DFS中的根目录。

最后点击一键安装完成GeaFlow安装部署。

## 构图
### 一次构图
GeaFlow可以支持TB级别的超大规模图，使得用户构图完成后，轻松应对业务演进。 超大规模数据的存储需要DFS的支持，数据来源可以是数据库、Hive、Kafka等等任何外部系统，通过对应的Connector读写数据。

举例来说，我们创建实现商业智能应用的第一张图，命名为bi。 创建图后， 将外部数据源的业务数据导入图中，使用对应的Connector完成数据导入。

![image](../../../../assets/images/posts/20230715/create-graph.png)

图的Schema定义如下，图名称为bi：
```sql
CREATE GRAPH IF NOT EXISTS bi (
  Vertex Tag (id bigint ID, name varchar, url varchar),
  Vertex Person (id bigint ID,  creationDate bigint,  firstName varchar,  lastName varchar,
                 gender varchar,  browserUsed varchar,  locationIP varchar),
  Vertex Post (id bigint ID,  creationDate bigint,  browserUsed varchar,  locationIP varchar,
               content varchar, length bigint,  lang varchar, imageFile varchar),
  Edge hasType (srcId bigint SOURCE ID, targetId bigint DESTINATION ID),
  Edge hasTag (srcId bigint SOURCE ID,  targetId bigint DESTINATION ID),
  Edge hasInterest (srcId bigint SOURCE ID, targetId bigint DESTINATION ID),
  Edge hasCreator (srcId bigint SOURCE ID,  targetId bigint DESTINATION ID),
  Edge knows (srcId bigint SOURCE ID, targetId bigint DESTINATION ID, creationDate bigint)
) WITH (
	storeType='rocksdb'
);

```
### 追加
即使在构图完成后，也可以向图中追加新的数据。 追加数据时，还可以流式触发增量查询，不断更新查询结果。

![image](../../../../assets/images/posts/20230715/insert.png)

向图bi中单独追加Person和knows点边数据的GQL：
```sql
Create Table If Not Exists tbl_Person (id bigint, type varchar, creationDate bigint, firstName varchar,
    lastName varchar, gender varchar, browserUsed varchar, locationIP varchar)
WITH ( type='file', geaflow.dsl.file.path='/tmp/data/bi_person');
INSERT INTO bi.Person
SELECT id, creationDate, firstName, lastName, gender, browserUsed, locationIP FROM tbl_Person;

Create Table If Not Exists tbl_edge_va (srcId bigint, targetId bigint, type varchar, va bigint)
WITH ( type='file', geaflow.dsl.file.path='/tmp/data/bi_edge_with_value');

INSERT INTO bi.knows SELECT srcId, targetId, va FROM tbl_edge_va WHERE type = 'knows';
```

## 图研发实现商业智能
### 数据调查

分析商业数据的第一步是明确问题，并通过数据调查开始针对性地进行数据分析，避免浪费时间和资源在无用的分析上。 我们已经利用GeaFlow将数据导入图bi中，可以通过运行图查询作业快速地得出结论和洞察。

以如下查询为例，它帮助我们了解用户关注的tag都有哪些，结果被写入本地或DFS的interest_tag文件夹中。

```sql
CREATE TABLE IF NOT EXISTS tbl_interest_tag
(personId bigint, personName varchar, tagId bigint, tagName varchar)
WITH (
	type='file',
	geaflow.dsl.file.path='/tmp/geaflow/chk/result/interest_tag'
);

USE GRAPH bi;

INSERT INTO tbl_interest_tag
MATCH (person:Person)-[:hasInterest]->(tag:Tag)
RETURN person.id as personId, concat(person.lastName, person.firstName) as personName,
       tag.id as tagId, tag.name as tagName
ORDER BY personId, tagId LIMIT 100
;
```

其核心查询是 **MATCH (person:Person)-[:hasInterest]->(tag:Tag)**。 这里**()**表示查询图中的点，**[]**表示查询图中的边。 完整的含义是"用户感兴趣的标签"，GeaFlow采用类似ISO-GQL的模式表达，可以方便自然地描述关系。

通过管控平台Console，分析人员可以提交一系列研究作业。 这些图查询作业会通过GeaFlow引擎自动提交到K8S集群中分布式地运行，大大太高了数据分析的能力和效率。

![image](../../../../assets/images/posts/20230715/jobs.png)

运行的图查询均可在Console界面查看，方便回溯和管理。

### 图研发

通过了基础的数据调查，分析师开始关注用户在标签上展现的中心性。 我们可以自定义一种中心性关系：

**用户相对标签的中心性包括两部分，第一部分等于用户发送该标签消息数，第二部分为用户对该TAG感兴趣则+100**

计算图中所有用户对TAG的中心性可以被表示为如下的查询：

```sql
CREATE TABLE IF NOT EXISTS tbl_centrality_score
(personId bigint, personName varchar, tagId bigint, tagName varchar, personCentralityScore bigint)
WITH (
	type='file',
	geaflow.dsl.file.path='/tmp/geaflow/chk/result/tag_centrality_score'
);

USE GRAPH bi;

INSERT INTO tbl_centrality_score
MATCH (interest_tag:Tag)<-[:hasInterest]-(person:Person)
LET person.hasInterest = COUNT((person:Person)-[:hasInterest]->(tag where id = interest_tag.id) => tag.id)
LET person.messageScore = COUNT((person:Person)
                                <-[:hasCreator]-(message:Post|Comment)
                                -[:hasTag]->(tag where id = interest_tag.id)
                                => tag.id)
LET person.score = person.messageScore + IF(person.hasInterest > 0, 100, 0)
RETURN person.id as personId, concat(person.lastName, person.firstName) as personName,
       interest_tag.id as tagId, interest_tag.name as tagName, person.score as personCentralityScore
ORDER BY personId, personCentralityScore DESC LIMIT 100
;
```

其中**person.score**由**person.messageScore**和**IF(person.hasInterest > 0, 100, 0)**两部分组成，返回的列表经分数降序排列，取前100条记录。

更近一步地，分析人员可以定义这种中心性分数的一度传播，使其成为中介中心性分数的近似值。 我们设计的查询关系如下图，用户对某个TAG的中介中心性分数近似为与其具有knows关系用户的中心性分数之和。

![image](../../../../assets/images/posts/20230715/bi_08_graph.png)

表示为GQL查询如下：

```sql
CREATE TABLE IF NOT EXISTS tbl_person_centrality (
  personId bigint,
  score bigint,
  friendsScore bigint
) WITH (
	type='file',
	geaflow.dsl.file.path='/tmp/geaflow/chk/result/person_centrality'
);

USE GRAPH bi;

INSERT INTO tbl_person_centrality
MATCH (person:Person)
--tag.name = 'Huang Bo' tag.id = 1020002
LET person.hasInterest = COUNT((person:Person)-[:hasInterest]->(tag where id = 1020002) => tag.id)
LET person.messageScore = COUNT((person:Person)
                                <-[:hasCreator]-(message:Post|Comment
                                where creationDate > 1672502400000 and creationDate < 1696160400000)
                                -[:hasTag]->(tag where id = 1020002)
                                => tag.id)
LET GLOBAL person.score = person.messageScore + IF(person.hasInterest > 0, 100, 0)
MATCH (person:Person)-[:knows]-{0,1}(friend:Person)
RETURN person.id as personId, person.score as personCentralityScore,
SUM(IF(friend.id = person.id, CAST(0 as BIGINT), CAST(friend.score as BIGINT))) as friendScore
GROUP BY personId, personCentralityScore
ORDER BY personCentralityScore + friendScore DESC, personId LIMIT 100
;
```

这个查询计算了一段时间内，**1020002**这个标签相关的用户一度中介中心性分数，把个人的中心性分数与中介中心性分数相加，降序排列后输出前100条记录。

至此，我们模拟进行了一次图研发过程，得到了一种可以利用的标签中心性计算方法。 整个过程都在GeaFlow的管控平台Console中完成，分析人员无需关注云上的作业运行细节。

### 构建应用

如果我们希望长期利用图研发得到的计算方法，则可以将其构建为长期运行的图计算应用。

通过接入数据源进行流式图查询，GeaFlow将在图每次更新或外部数据触发时，执行一次中介中心性分数计算。 更新的结果将被输出到外部系统，方便分发给下游。

搭建方法可以参考"[谁在以太坊区块链上循环交易？TuGraph+Kafka的0元流图解决方案](../../06/16/谁在以太坊区块链上循环交易-TuGraph+Kafka的0元流图解决方案.html)"这篇博文，这里不再赘述。

## 总结
本文介绍了GeaFlow如何在云原生的K8S环境中安装部署，并模拟了一次商业智能研究过程。 全程采用GeaFlow自有的管控平台Console提交作业，展现了系统强大的表达和计算能力。

GeaFlow(品牌名TuGraph-Analytics) 已正式开源，欢迎大家关注！！！

欢迎给我们 Star 哦!

Welcome to give us a Star!

GitHub:[https://github.com/TuGraph-family/tugraph-analytics](https://github.com/TuGraph-family/tugraph-analytics)

## 微信群
![image](../../../../assets/images/wechat.png)

请点击项目链接下方微信二维码添加微信用户群：[https://github.com/TuGraph-family/tugraph-analytics](https://github.com/TuGraph-family/tugraph-analytics)

