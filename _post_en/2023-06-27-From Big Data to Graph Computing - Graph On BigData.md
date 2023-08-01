---
layout: post
read_time: true
show_date: true
show_author: true
title: "From Big Data to Graph Computing - Graph On BigData"
date: 2023-06-27
tags: [大数据, 图计算, TuGraph, GeaFlow, 开源, github]
category: opinion
author: 彭志伟
description: "From Big Data to Graph Computing - Graph On BigData"
---

## Background

Since the publication of Google's three classic papers in the field of big data, GFS, MapReduce, and BigTable, in 2003, the field of big data has made significant progress. Especially in the field of open-source big data, various excellent open-source big data engine projects have emerged one after another, including Hadoop, Hive, Storm, Spark, Flink, and Presto. These projects cover various computing forms such as offline computation, stream computation, OLAP queries, and hybrid stream-batch computation, and the processing technologies for big data are becoming more and more advanced and diversified.

These big data engines mainly handle tabular data, that is, the data to be processed is modeled using a tabular model and then processed. Although the tabular model is relatively simple and easy to understand, it has its limitations, especially in dealing with complex relationship operations and expressions. The tabular model mainly uses the Join method to handle the relationship between tables, which introduces a large amount of shuffle and increases running resources. This disadvantage of the Join method becomes more obvious when the degree of association is high. In addition, it is difficult to express complex relationships such as shortest paths and k-hops using the SQL language in the tabular model.

As a data model defined by points and edges, the graph model can naturally describe relationships. In the graph model, points represent entities and edges represent relationships. For example, in a social network graph, each person can be represented by a point, and the relationships between people are represented by edges. There can be various complex relationships between people, which can be represented by different types of edges. Based on the graph model, complex relationships and their operations can be well described. In addition, the storage model of the graph naturally stores the relationships between points and edges, and better computational performance can be obtained at the computation level.

![image](../../../../assets/images/posts/20230627/zhuanzhang.png)

## Real-time Graph Computing Engine - GeaFlow

In the context of Ant Financial's risk control, there are a large number of complex relationship processing tasks, such as finding multi-hop transfer relationships in the anti-money laundering system to check for loops and determine whether users are engaged in money laundering activities; in the log attribution analysis scenario, it is necessary to analyze the user's behavioral path, etc. These scenarios involve complex relationship processing and have high requirements for real-time computation. The business often requires delays of minutes or even seconds. At the same time, the graph data scale can reach billions or even trillions of points and edges. Traditional big data engines cannot meet the above requirements. For example, Spark GraphX has the ability to process large-scale graph data, but it mainly focuses on offline computation scenarios and cannot meet real-time requirements. Flink has powerful real-time computation capabilities, but it is difficult to handle real-time multi-hop Join association computations, especially for scenarios with large data scales.

In the face of these problems and challenges, the graph computing team at Ant Financial has developed a distributed real-time graph computing engine called GeaFlow (branded as TuGraph-Analytics) through years of exploration and practice. GeaFlow uses the graph model as the basic data model and defines a set of graph computing programming interfaces on top of the graph model. It combines with stream processing capabilities to achieve real-time graph computing. At the DSL language level, GeaFlow combines the table processing language SQL with the graph query language ISO/GQL to achieve the data analysis capabilities of graphs and tables. Through the stream graph computing capabilities of GeaFlow, the problem of real-time computation of large-scale data with complex relationship associations in the financial scenario is well solved.

## GeaFlow Overall Architecture

The GeaFlow overall architecture includes the following layers from top to bottom:

![image](../../../../assets/images/posts/20230627/geaflow_arch.png)

* GeaFlow DSL: GeaFlow provides users with a language for combining graph and table analysis, using SQL + ISO/GQL notation. Users can write real-time graph computation tasks in a similar way to programming in SQL.
* GraphView API: GeaFlow defines a set of graph computing programming interfaces based on GraphView as the core, including graph construction, graph computation, and Stream API interfaces.
* GeaFlow Runtime: GeaFlow runtime includes GeaFlow graph operators, task scheduling, failover, shuffle, and other core functionalities.
* GeaFlow State: GeaFlow's graph state storage is used to store the point and edge data of the graph. The state of stream computation, such as aggregation state, is also stored in the State.
* K8S Deployment: GeaFlow supports deployment and operation using the K8S platform.
* GeaFlow Console: GeaFlow's control platform includes job management, metadata management, and other functions.

## Integration of GeaFlow with the Big Data Ecosystem
A graph computing system is not an isolated system; it must be integrated with the existing big data ecosystem to better solve problems in the field of big data. GeaFlow supports integration with mainstream big data ecosystems through Connector plugins, such as Kafka/Hive/HDFS. With the Connector plugin, it is easy to integrate data from the big data ecosystem into the graph computing system. In the following example, we will use Hive to illustrate how to import data from a data warehouse into the GeaFlow graph storage and run a graph algorithm.

### Graph Definition

First, we need to define a graph using the Create Graph syntax:
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
  shardCount = 1
);
```
This graph definition includes the point table person and the edge table knows. The point table person defines the attribute information of the point and the id field, which uniquely identifies the points in the graph and serves as the primary key of the point table, defined using the ID keyword. The edge table knows defines the friend relationship, where srcId is the starting point of the relationship, defined using the SOURCE ID keyword, and targetId is the target point of the relationship, defined using the DESTINATION ID keyword. The weight field is an attribute field of the edge. A graph can have zero or more attribute fields in its point or edge table.

### Hive Table Definition

First, we need to define a Hive point table and a Hive edge table, specifying the schema information and metastore URI of the tables:

```sql
set geaflow.dsl.window.size = -1;

CREATE TABLE IF NOT EXISTS hive_person (
  id BIGINT,
  name VARCHAR
) WITH (
  type='hive',
  geaflow.dsl.hive.database.name = 'default',
  geaflow.dsl.hive.table.name = 'user',
  geaflow.dsl.hive.metastore.uris = 'thrift://localhost:9083'
);

CREATE TABLE IF NOT EXISTS hive_knows (
  src_id BIGINT,
  target_id BIGINT,
  weight DOUBLE
) WITH (
  type='hive',
  geaflow.dsl.hive.database.name = 'default',
  geaflow.dsl.hive.table.name = 'relation',
  geaflow.dsl.hive.metastore.uris = 'thrift://localhost:9083'
);
```

GeaFlow is a stream graph computing engine, and the data source is divided into a series of windows according to the window size, and the engine processes the data of these windows in sequence. If the window size is set to -1, it represents an "All Window," which means processing all data in one go. For batch data sources like Hive, the window size needs to be set to -1 for processing.

### Graph Construction

Graph construction is the process of writing external data table data into the graph. This can be done using INSERT statements. The following statements will insert the data from the Hive tables into the person and knows tables of the friend graph, completing the construction of the graph data.

```sql
INSERT INTO friend.person(id, name)
SELECT
 id, name
FROM hive_person
;

INSERT INTO friend.knows
SELECT src_id, target_id, weight * 10
FROM hive_knows
;
```

### Graph Computation

Next, we need to perform graph algorithm computations on the constructed graph data. Let's take Single-Source Shortest Path (SSSP) as an example:

```sql
CREATE TABLE IF NOT EXISTS result (
  vid int,
	distance bigint
) WITH (
	type='file',
  `geaflow.file.persistent.config.json` = '{\'fs.defaultFS\':\'namenode:9000\'}',
	geaflow.dsl.file.path='/path/to/result'
);
-- Define the graph to be used for computation
USE GRAPH friend;

INSERT INTO result
CALL SSSP(1) YIELD (vid, distance)
RETURN vid, distance
;
```

First, we need to define a result table to store the computation results, and then use the USE GRAPH command to set the current graph for computation. Finally, use the CALL statement to execute the SSSP algorithm (where the parameter of the SSSP algorithm is the starting point ID) and write the computation results to the result table.

# Conclusion

This article first introduces the historical background of the graph computing engine GeaFlow, and then explains how GeaFlow integrates with the big data ecosystem. Through an example, we demonstrate how to transform data from Hive into a graph and run an SSSP algorithm on the graph.


