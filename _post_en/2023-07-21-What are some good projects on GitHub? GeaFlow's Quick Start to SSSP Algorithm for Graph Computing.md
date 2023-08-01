---
layout: post
read_time: true
show_date: true
show_author: true
title: "What are some good projects on GitHub? GeaFlow's Quick Start to SSSP Algorithm for Graph Computing"
date: 2023-07-21
tags: [蚂蚁集团, 图计算, 单源最短路径, GeaFlow, 开源, github]
category: news
author: TuGraph
description: "This article introduced the basic principles of the graph algorithm SSSP supported by the real-time graph computing engine GeaFlow, as well as its implementation details in GeaFlow, and demonstrated an application on the GitHub dataset."
---

Tips: This article has been translated from Chinese to English by ChatGPT 3.5. We apologize for any inaccuracies.

## Introduction
The following image is a network graph of approximately 500 open source project repositories and topics on GitHub. The dense web of connections makes it difficult for anyone to discern any useful information. However, GitHub currently has over 3 million repositories in total!

![image](../../../../assets/images/posts/20230721/github1.png)

How to discover interesting projects in 5 minutes? 

Today, we will use GeaFlow to help us implement SSSP (single-source shortest path algorithm) and give it a try. 

GeaFlow (brand name TuGraph-Analytics) is an open-source distributed real-time graph computing engine developed by Ant Group, and is currently widely used in scenarios such as financial risk control, social networks, knowledge graphs, and data applications.

## Introduction to SSSP (Single Source Shortest Path Algorithm)

SSSP (Single Source Shortest Path) algorithm is a graph theory algorithm used to find the shortest path from a starting point to all other nodes. This algorithm can be applied to a variety of practical problems, such as map navigation and network topology.

In the relationship network of open source project repositories and topics on GitHub, the relationship edges from repositories to topics and back to repositories can support the operation of the SSSP algorithm.

![image](../../../../assets/images/posts/20230721/github2.png)

As shown in the partial relationship network, starting from the starting point, the number of arrows can be used to mark the distance from topics/repositories to the source. For example, the distance between the repositories FiraCode and Font-Awesome is 2, which can be reached through 2 arrows, and they are also the closest associated repositories to each other.

Simply put, if we mark the repositories we are interested in, the repositories that are closest to our interested repositories are recommended as good repositories. Or, to take it a step further, repositories that are closer and have more stars are more recommended.

## Implementing SSSP with GeaFlow

To run the SSSP algorithm, we can specify the graph to be used and directly call the graph algorithm in the graph query. The syntax is as follows:

```sql
USE GRAPH github_repo_topic
INSERT INTO tbl_result
CALL sssp('source_vertex') YIELD (repoName, distance)
RETURN repoName, distance;
```

This piece of code runs on the graph github_repo_topic, taking source_vertex as the starting point of the algorithm and outputting the distance to all other vertices. If fewer results are needed, a WHERE condition can be added to the RETURN statement for filtering, just like in SQL statements!

If a custom graph algorithm is needed, we can implement the AlgorithmUserFunction interface. GeaFlow has built-in generic implementations of various graph algorithms, so there is no need for separate customization. The reference implementation of the SSSP algorithm is shown below:

```java
@Description(name = "sssp", description = "built-in udga Single Source Shortest Path")
public class SSSP implements AlgorithmUserFunction<Object, Long> {

    private AlgorithmRuntimeContext<Object, Long> context;
    
    @Override
    public void init(AlgorithmRuntimeContext<Object, Long> context, Object[] parameters) {
        //Initialize the algorithm context.
        this.context = context;
    }

    @Override
    public void process(RowVertex vertex, Iterator<Long> messages) {
        long currentDistance;
        if (context.getCurrentIterationId() == 1L) {
            //Initialize the initial distance value for all vertices.
        } else if (context.getCurrentIterationId() <= $maxIteration) {
            //Calculate the shortest distance.
        } else {
            //Return the result.
        }
        //Update the distance value
        context.updateVertexValue(ObjectRow.create($currentDistance));
        //Send messages to neighbors
        context.sendMessage(vertex.getId(), $currentDistance);
        long scatterDistance = $currentDistance == Long.MAX_VALUE ? Long.MAX_VALUE : currentDistance + 1;
        for (RowEdge edge : context.loadEdges(EdgeDirection.OUT)) {
            context.sendMessage(edge.getTargetId(), scatterDistance);
        }
    }

    @Override
    public StructType getOutputType() {
        //Data type of the algorithm return value.
        return new StructType(
            new TableField("id", StringType.INSTANCE, false), 
            new TableField("distance", LongType.INSTANCE, false)
        );
    }
}
```

Graph queries are submitted as jobs to be completed, which can be run locally or on a K8S cluster. GeaFlow provides a console for managing and tracing these graph development jobs.
![image](../../../../assets/images/posts/20230721/github3.png)

## Blind men touch the elephant on the GitHub relationship graph

Without further ado, let's find out which projects on GitHub, with a distance of 2 (i.e. having common topics), are currently the most starred project on GitHub?

The most starred project currently on GitHub is freeCodeCamp, and here is the data [GitHub Public Repository Metadata](https://www.kaggle.com/datasets/pelmers/github-repository-metadata-with-5-stars) as of May 2023.

```sql
USE GRAPH github_repo_topic
INSERT INTO tbl_result
SELECT repoName, distance FROM (
    CALL sssp('freeCodeCamp') YIELD (repoName, distance)
    RETURN repoName, distance
) WHERE distance = 2 
LIMIT 10
;
```

In a short period of time, we got the calculation results. Let's take a look at the good projects that GeaFlow has found for me. Here we present 10 randomly selected records, not sorted by star count.

```txt
id,stars,forks
papers-we-love,72164,5162
system-design-primer,220197,39109
free-programming-books-zh_CN,102417,27516
33-js-concepts,56077,7850
build-your-own-x,201052,19629
30-seconds-of-code,111510,11483
carbon,32588,1854
freecodecamp.cn,36459,1369
Web-Dev-For-Beginners,69680,10904
free-programming-books,279431,55158
```

## Conclusion
This article introduced the basic principles of the graph algorithm SSSP supported by the real-time graph computing engine GeaFlow, as well as its implementation details in GeaFlow, and demonstrated an application on the GitHub dataset.



