---
layout: post
read_time: true
show_date: true
show_author: true
title: "图加速数据湖分析-TuGraph和Hudi集成"
date: 2023-07-12
tags: [Hudi, 图计算, 数据湖, GeaFlow, 开源, github]
category: news
author: TuGraph
description: "本文主要分析了表模型的现状和问题，然后介绍了图模型在处理关系运算上的优势，接着介绍了图计算引擎GeaFlow和数据湖格式hudi的整合，利用图计算引擎加速数据湖上的关系运算."
---
## 表模型现状与问题
关系模型自1970年由埃德加·科德提出来以后被广泛应用于数据库和数仓等数据处理系统的数据建模。关系模型以表作为基本的数据结构来定义数据模型，表为二维数据结构，本身缺乏关系的表达能力，关系的运算通过Join关联运算来处理。表模型简单且易于理解，在关系模型中被广泛使用。
随着互联网信息技术的发展，处理的数据规模越来越大，大数据系统应运而生。表模型作为重要的数据模型依然被Spark/Hive/Flink等主流大数据引擎所采用，表模型之上的SQL查询语言也被广泛使用在大数据分析处理中。然而随着应用场景的丰富和处理数据规模的变大，表模型的问题也越来越多的暴露出来。

- 首先，关系运算成本高

表模型本身缺乏关系描述能力，只能通过Join运算来完成关系的计算。无论在批处理系统里面还是流计算系统中，Join都是非常重的操作，需要大量的数据shuffle和计算开销，在流计算系统中，还需要存放左右两张流表的历史状态，存储消耗极高。

- 其次，数据冗余时效性低

数仓分析的场景为了提高数据查询性能，往往将多张表提前物化成一张大宽表。大宽表虽然可以加速查询性能，然而其数据膨胀和冗余非常严重。需要将多张表Join成一张表，表与表之间一对多的关联关系导致一张表的数据通过关联会放大多份，造成数据量指数级膨胀和冗余。而且宽表一旦生成，如需添加新的表进来，需要重新生成新宽表，计算开销大，不灵活。另外，基于Join方式成本高，基于宽表方式很难实现数仓实时化。

- 最后，无法支持复杂关系查询

基于SQL join方式很难描述复杂关系查询，比如查询一个人4度以内所有好友或者查询最短路径等。这些复杂关联关系通过SQL表的join方式很难描述。
## 图模型解决方案
### 图是关系的天然描述
图是对关系的一种天然描述，图模型是一种以点和边作为基本单元定义的数据模型天然可以描述关联关系。在图模型里面以点代表实体，以边代表关系。比如在人际关系图里面，每一个人可以用一个点来表示，人和人之间的关系通过边来表示，人与人之间可以存在各种各样的复杂关系，这些关系都可以通过不同的边来表示。所以图模型里面天然就包含关系，是对关系最自然的表达。

![image](../../../../assets/images/posts/20230712/g1.png)

### 图是关系的物化
图模型中本身包含点边关系的定义，在数据存储层面会按照点边关系存放数据，点和其邻边会存储在一起。所以图存储层面对关系做了物化，相比与表的Join方式，可以获取更好的关联计算性能。相比宽表的关系物化方式，由于图结构本身的点边聚合性，不会出现宽表展开导致的数据膨胀，其存储空间会更小，如下图所示。

![image](../../../../assets/images/posts/20230712/g2.png)

### 图加速数据查询
利用图的关系物化的能力，可以加速关系运算的查询，如下例子：
学生、课程和教师三个实体表，实体之间存在选课(selectCourse)、考试(examination)和教学(teach)三种关系.这些实体之间的关系查询可以通过图查询来表示。

![image](../../../../assets/images/posts/20230712/g3.png)

- 查询数学考试成绩前十的学生及其分数
```sql
select s.id, s.name, e.score 
from student s join examination e join c course
on s.id = s.student_id and e.course_id = c.id
where c.name = 'math'
order by e.score desc limit 10
```
对应图查询：
```sql
Match(s:student) - [e:examination]->(c:course where c.name = 'math')
Return s.id, s.name, e.score order by e.score desc limit 10
```

- 查询选课人数最多的老师Top 3
```sql
select tr.id, count(s.id) 
from student s join selectCourse sc join course c 
join teach th join teacher tr
on s.id = sc.student_id and sc.course_id = c.id and c.id = th.course_id
and th.teacher_id = tr.id
order by count(s.id) desc
limit 3
```
对应图查询：
```sql
Match(s:student)-[sc:selectCourse]->(c:course)<-[th:teach]-(tr:teacher)
Return tr.id, count(s.id) as cnt order by cnt desc limit 3
```

### 图模型和表模型关系
图模型本身包含点数据集和边数据集，点边数据集来源于表数据，比如Hive表、Hudi表等。所以，图模型是表模型的超集，是对表模型的补充和完善。图模型可以起到类似宽表的作用，物化表的关系，同时能更灵活的定义关系，消耗更小的存储开销.

## GeaFlow和Hudi集成
GeaFlow(品牌名TuGraph-Analytics)是蚂蚁自研的分布式实时图计算引擎，兼顾离线图计算能力。GeaFlow以图模型作为基本的数据模型，在图模型基础之上定义了一套图计算的编程接口，同时和流式处理能力相结合，实现了流式图计算的能力。在DSL语言层面，GeaFlow将表处理语言SQL和图查询语言ISO/GQL相结合，实现了图表一体的数据分析能力。通过GeaFlow图计算的能力，很好的解决了大规模数据关联关系计算的问题。

Hudi是业界热门的数据湖格式，旨在解决数据湖中数据的变更管理问题。Hudi使用了一种基于日志的存储方式，可以支持数据的实时增量、删除和更新，并且能够保证数据的一致性和可靠性。Hudi的核心思想是将数据划分成小的数据块，每个数据块都包含了数据的变更历史，可以通过增量方式和全量方式读取和写入数据。Hudi支持多种数据格式，包括Parquet、ORC、CSV等，并且可以与Hadoop、Spark、Flink等大数据处理框架无缝集成，可用于数据湖的建设和数据管理。Hudi的出现大大简化了数据湖的数据变更管理和数据处理流程，是一个非常优秀的数据管理框架。

GeaFlow支持和多种数据源集成，包括Hudi。利用GeaFlow图计算的能力，可以对Hudi数据湖数据做关系物化，加速DWD层的查询性能和时效性，同时也可以基于图数据做更多复杂的图算法分析。

![image](../../../../assets/images/posts/20230712/g4.png)

以下为GeaFlow使用Hudi构图，然后进行4度环路查找和SSSP算法计算的例子：
### 图定义
我们首先需要定义张图，使用Create Graph语法定义如下：
```sql
CREATE GRAPH IF NOT EXISTS friend (
  Vertex person (
    id bigint ID,
    name varchar
  ),
  Edge knows (
    srcId bigint SOURCE ID,
    targetId bigint DESTINATION ID,
    weight double
  )
) WITH (
  storeType='rocksdb',
  shardCount = 4
);
```
这张图定义包含点表person和边表knows. 点表person定义了点的属性信息和id字段，id字段唯一标识图里面的点，为点表的主键，通过**ID **关键字来定义。边表knows里面定义好友关系，srcId为关系的起点，通过**SOURCE ID**关键字定义；targetId为关系的目标点，通过**DESTINATION ID**关键字定义。weight字段则为边的一个属性字段。一张图的点边或者边表可以包含零个或者多个属性字段。
### Hudi表定义
首先我们需要定义一张Hudi点表和Hudi边表：
```sql
set geaflow.dsl.window.size = -1;

CREATE TABLE IF NOT EXISTS hudi_person (
  id BIGINT,
  name VARCHAR
) WITH (
   type='hudi', -- hdfs 配置，也可通过HADOOP_HOME环境变量获取
  `geaflow.file.persistent.config.json` = '{\'fs.defaultFS\':\'namenode:9000\'}',
	geaflow.dsl.file.path='/path/to/hudi_person'
);

CREATE TABLE IF NOT EXISTS hudi_knows (
  src_id BIGINT,
  target_id BIGINT,
  weight DOUBLE
) WITH (
  type='hudi', -- hdfs 配置，也可通过HADOOP_HOME环境变量获取
  `geaflow.file.persistent.config.json` = '{\'fs.defaultFS\':\'namenode:9000\'}',
	geaflow.dsl.file.path='/path/to/hudi_knows'
);
```
GeaFlow是一个流式图计算引擎，数据源按照window size切分成一系列的window, 引擎会依次处理这些window的数据。如果window size设置为-1，则代表一个All Window，即一次全量处理所有数据。对于Hudi这样的批数据源接口，需要设置window size为-1来处理。
### 构图
构图是将外部数据表的数据写入到图里面，可以通过Insert语句来完成。如下语句，分布将hudi表里面的数据写入到friend图的person表和knows表里面，完成图数据的构建。
```sql
INSERT INTO friend.person(id, name)
SELECT
 id, name
FROM hudi_person
;

INSERT INTO friend.knows
SELECT src_id, target_id, weight * 10
FROM hudi_knows
;
```
### 图计算
接下来是对构建好的图数据做图计算，我们以SSSP(单源最短路径)和四度环路检查为例进行介绍：
```sql
CREATE TABLE IF NOT EXISTS sssp_result (
  vid int,
	distance bigint
) WITH (
	type='file',
  `geaflow.file.persistent.config.json` = '{\'fs.defaultFS\':\'namenode:9000\'}',
	geaflow.dsl.file.path='/path/to/result'
);
-- 定义计算使用的图
USE GRAPH friend;

INSERT INTO sssp_result
CALL SSSP(1) YIELD (vid, distance)
RETURN vid, distance
;

-- 图算法执行
INSERT INTO result
CALL SSSP(1) YIELD (vid, distance)
RETURN vid, distance
;
```
首先需要定义一个结果表result来存放计算结果，然后通过**USE GRAPH**命令来设置当前计算用到的图。最后通过CALL语句来执行SSSP算法(其中SSSP算法的入参为起始点id), 并将计算结果写入结果表。
四度环路匹配如下语句，通过Match匹配一个4度环路的pattern,然后将结果写出结果表.
```sql
CREATE TABLE IF NOT EXISTS match_result (
  a_id bigint,
  b_id bigint,
  c_id bigint,
  d_id bigint,
  a1_id bigint
) WITH (
  type='file',
  `geaflow.file.persistent.config.json` = '{\'fs.defaultFS\':\'namenode:9000\'}',
	geaflow.dsl.file.path='/path/to/result2'
);
-- 四度环路匹配
INSERT INTO match_result
SELECT
  a_id,
  b_id,
  c_id,
  d_id,
  a1_id
FROM (
  MATCH (a:person) -[:knows]->(b:person) -[:knows]-> (c:person)
   -[:knows]-> (d:person) -> (a:person)
  RETURN a.id as a_id, b.id as b_id, c.id as c_id, d.id as d_id, a.id as a1_id
);
```
## 总结
本文主要分析了表模型的现状和问题，然后介绍了图模型在处理关系运算上的优势，接着介绍了图计算引擎GeaFlow和数据湖格式hudi的整合，利用图计算引擎加速数据湖上的关系运算.

GeaFlow(品牌名TuGraph-Analytics) 已正式开源，欢迎大家关注！！！

欢迎给我们 Star 哦!

Welcome to give us a Star!

GitHub👉[https://github.com/TuGraph-family/tugraph-analytics](https://github.com/TuGraph-family/tugraph-analytics)

## 微信群
![image](../../../../assets/images/wechat.png)

请点击项目链接下方微信二维码添加微信用户群：[https://github.com/TuGraph-family/tugraph-analytics](https://github.com/TuGraph-family/tugraph-analytics)




