---
layout: post
read_time: true
show_date: true
show_author: true
title: "TuGraph Analytics' Stream Graph Computing Paper Selected for International Top Conference SIGMOD"
date: 2023-06-21
tags: [TuGraph-Analytics, 流式图计算, SIGMOD, 论文, 数据库]
category: news
author: TuGraph
description: "TuGraph Analytics' Stream Graph Computing Paper Selected for International Top Conference SIGMOD"
---

From June 18th to 23rd, 2023, the 2023 ACM SIGMOD International Conference on Database was held in Seattle, USA, and a paper from the Ant Stream Graph Computing team was selected. 

ACM SIGMOD International Conference on Data Management is initiated by the Data Management Committee (SIGMOD) of the American Computer Association (ACM), and is one of the three top academic conferences in the database industry, along with VLDB and ICDE. Its collection of papers represents the highest level in the field of databases and is an important indicator of future database technology development. 

The Ant Stream Graph Computing team's paper, "GeaFlow: A Graph Extended and Accelerated Dataflow System," was included in SIGMOD 2023, indicating that the team's achievements in stream graph computing not only have extensive applications in the industry but are also further recognized in the academic community.

![image](../../../../assets/images/posts/20230621/tu1.png)

Image caption: GeaFlow is the internal code name for the Ant Stream Graph Computing engine used by TuGraph Analytics. This article will continue to use "GeaFlow" as used in the paper, making it easier for readers to follow. 

The project code for the image can be found at: [https://github.com/TuGraph-family/tugraph-analytics](https://github.com/TuGraph-family/tugraph-analytics)


## Background

With the continuous development of stream processing technology, it is widely used in various application fields such as monitoring, recommendation, and advertising. Although existing stream processing systems have been proven to be very effective in handling set queries of window types, they face many challenges when dealing with more complex data structures, such as graph-structured data.

Firstly, in stream computing, join is a very expensive operation, especially for dynamically changing data, as each incoming data needs to find its corresponding data from another stream, and it also involves data materialization and shuffle. In stream graph computing, hundreds or thousands of graph iterations are usually required to obtain results, corresponding to hundreds or thousands of join operations in stream computing.

Secondly, the intermediate states of data aggregation in stream computing usually need to maintain a much smaller amount of data than the original data size of the window in a fixed or sliding window, and the window usually is not particularly large. In stream graph computing, it usually needs to maintain a long and dynamically changing graph window, such as in risk identification, which requires backtracking to several months or even years of data.

To solve these problems, the Ant Stream Graph Computing team has developed and designed a stream graph computing engine, GeaFlow, which can well meet the demands of stream dynamic graph analysis, traversal, and calculation, and also supports long window graph data state management, DSL-based stream graph task development, and other features.

GeaFlow has been widely used in Ant's risk control, marketing, and other scenarios, and has also performed well in the Double Eleven promotion. Now let's analyze the running scenario and internal technology of Geaflow.

## Running Scenario

Unlike graph databases, GeaFlow solves complex graph analysis and computation problems in high-throughput scenarios, such as deep graph traversal, subgraph matching, and full graph analysis. GeaFlow uses a stream event-driven approach for computation and queries, while utilizing query optimization techniques for query merging and acceleration. Below, we describe the running scenario of GeaFlow.

![image](../../../../assets/images/posts/20230621/tu2.png)

<center>Figure 1</center>

Figure 1 describes a typical GeaFlow data flow diagram. From the perspective of the overall pipeline, it performs stream graph construction, and the data for graph construction can include various types of data, such as message queues or log systems. The data can be processed by ETL and finally converted into points and edges written into the graph storage, which periodically performs checkpointing to ensure ExactlyOnce.

Simultaneously, it also drives graph computation based on incremental and changing data, such as user IDs constantly making payments in the anti-fraud scenario. Unlike other general-purpose stream computing systems, since graph computation is iterative, GeaFlow supports iterative graph processing on stream links. Finally, the output data of graph computation can be processed, such as Window TopN computation and output in the figure above.

## Architecture Design

To support the above scenarios, GeaFlow has made the following architecture design:

![image](../../../../assets/images/posts/20230621/tu3.png)

<center>Figure 2</center>

As shown in Figure 2, compared to typical stream systems, we have made the following extensions and supplements. The overall architecture includes the following layers from top to bottom:

### Hybrid DSL
GeaFlow innovatively integrates table and graph semantics, using SQL for table DSL and Gremlin for graph DSL to describe real-time graph computing tasks. Users can easily write real-time graph computing tasks through programming similar to SQL. Meanwhile, in query planning, we optimize graph semantics for logical execution plans and physical execution plans, and optimize multiple query statements by merging queries and other methods.

### Core API
GeaFlow has designed a set of operators with state semantics, supporting multiple stream and graph semantics, including standard stream operators and vertex/edge-centric graph operators. In graph operators, we have established a good programming model, supporting various graph structures of users, and introduced session to handle concurrent graph queries.

### Streaming Engine
GeaFlow's distributed DAG execution layer at runtime can flexibly switch between stream and graph semantics, and efficiently support multi-query optimization. In the execution process, we use off-heap memory management, sharing graph state and batch execution technologies to optimize, greatly improving the overall throughput.

### GeaFlow State
GeaFlow's state storage is used to store and access stream and graph data. It can use KV semantic states, or graph semantic states. In the graph state, we have designed an efficient key layout to accelerate vertex and edge access, compatible with various multi-model backends. At the same time, we have also developed a graph-native backend, which supports more efficient graph access and flexible scaling with a class CSR index structure and a calculation and storage separation architecture.

## Experimental Results

Here we list the comparison between GeaFlow and other stream computing engines.

In the K-Hop Neighborhood scenario, we selected the LiveJournal and Twitter-2010 public datasets to compare GeaFlow, Flink, and Differential Dataflow. The experiment was conducted on a cluster of 8 containers, each with a specification of 8c16G.

Through implementation, we can see that：

In multiple hop traversal scenarios, GeaFlow's entire memory remains stable and does not increase significantly with the depth of iteration, while other general-purpose stream computing engines will have significant storage space amplification, resulting in memory overflow when the number of hops is too high.

GeaFlow significantly improves throughput compared to general-purpose routing computation engines that use Join to achieve it in typical stream graph scenarios.

![image](../../../../assets/images/posts/20230621/tu4.png)

<center>Figure 3</center>

In the graph query simulation scenario, we verified the relationship between query time and the number of queries executed concurrently.

In Figure 4, we can see that as the amount of concurrent query data in the batch increases, the increase in execution time is not particularly significant. It can be found that the time to merge 40 queries only increased by 22.7%, achieving a 32.6-fold speedup effect.

![image](../../../../assets/images/posts/20230621/tu5.png)


<center>Figure 4</center>

## Summary and Planning

GeaFlow originated from our attempt in Ant Financial's business scenario six years ago. At that time, we found that the existing stream processing architecture was not suitable for stream graph processing. After two years of incubation, GeaFlow successfully supported Ant Financial's anti-fraud business and achieved high throughput of processing over 10 million events per second during the Double Eleven period in 2018. The business effects brought by stream graph queries and analysis in these businesses were also very significant, proving that stream graph processing can provide a lot of help in financial scenarios. Later, we introduced DSL support to further reduce user development costs. We chose the combination of SQL + Gremlin and continuously improved our query optimizer. As a result, a large number of users began to use our DSL to query and analyze their graph computing scenarios. Today, GeaFlow has become one of Ant Financial's most popular computing engines.

Next, we plan to further develop GeaFlow in the following areas:

* Integration of more graph semantics, such as graph computation operators with incremental semantics.

* Explore more declarative graph query languages, such as OpenCypher, GQL, and SQL/PGQ, and incorporate CBO optimization, automatic tuning, and other capabilities.

* Explore the use of languages such as Rust/C++ to rewrite our execution engine to further improve performance.

