---
layout: post
read_time: true
show_date: true
show_author: true
title: "TuGgraph-Analytics图计算快速上手之紧密中心度算法"
date: 2023-09-20
tags: [图计算, 紧密中心度, 图算法, TuGraph-Analytics, 快速上手]
category: opinion
author: 李洁峰
description: "TuGraph-Analytics图计算快速上手之紧密中心度算法"
---
作者：张武科

## 概述
**紧密中心度（Closeness Centrality）**计量了一个节点到其他所有节点的紧密性，即该节点到其他节点的距离的倒数；节点对应的值越高表示紧密性越好，能够在图中传播信息的能力越强，可用以衡量信息流入或流出该节点的能力，多用与社交网络中关键节点发掘等场景。

## 算法介绍
对于图中一个给定节点，紧密性中心性是该节点到其他所有节点的最小距离和的倒数：
![image.png](../../../../assets/images/posts/20230920/tu0.png)

其中，u表示待计算紧密中心度的节点，d(u, v)表示节点u到节点v的最短路径距离；实际场景中，更多地使用归一化后的紧密中心度，即计算目标节点到其他节点的平均距离的倒数：
![image.png](../../../../assets/images/posts/20230920/tu1.png)

其中，n表示图中节点数。



## 算法实现
首先，基于`AlgorithmUserFunction`接口实现`ClosenessCentrality`，如下：

```java
@Description(name = "closeness_centrality", description = "built-in udga for ClosenessCentrality")
public class ClosenessCentrality implements AlgorithmUserFunction<Long, Long> {

    private AlgorithmRuntimeContext context;
    private long sourceId;

    @Override
    public void init(AlgorithmRuntimeContext context, Object[] params) {
        this.context = context;
        if (params.length != 1) {
            throw new IllegalArgumentException("Only support one arguments, usage: func(sourceId)");
        }
        this.sourceId = ((Number) params[0]).longValue();
    }

    @Override
    public void process(RowVertex vertex, Iterator<Long> messages) {
        List<RowEdge> edges = context.loadEdges(EdgeDirection.OUT);
        if (context.getCurrentIterationId() == 1L) {
            context.sendMessage(vertex.getId(), 1L);
            context.sendMessage(sourceId, 1L);
        } else if (context.getCurrentIterationId() == 2L) {
            context.updateVertexValue(ObjectRow.create(0L, 0L));
            if (vertex.getId().equals(sourceId)) {
                long vertexNum = -2L;
                while (messages.hasNext()) {
                    messages.next();
                    vertexNum++;
                }
                // 统计节点数
                context.updateVertexValue(ObjectRow.create(0L, vertexNum));
                // 向邻接点发送与目标点距离
                sendMessageToNeighbors(edges, 1L);
            }
        } else {
            if (vertex.getId().equals(sourceId)) {
                long sum = (long) vertex.getValue().getField(0, LongType.INSTANCE);
                while (messages.hasNext()) {
                    sum += messages.next();
                }
                long vertexNum = (long) vertex.getValue().getField(1, LongType.INSTANCE);
                // 记录最短距离和
                context.updateVertexValue(ObjectRow.create(sum, vertexNum));
            } else {
                if (((long) vertex.getValue().getField(1, LongType.INSTANCE)) < 1) {
                    Long meg = messages.next();
                    context.sendMessage(sourceId, meg);
                    // 向邻接点发送与目标点距离
                    sendMessageToNeighbors(edges, meg + 1);
                    // 标记该点已向目标点发送过消息
                    context.updateVertexValue(ObjectRow.create(0L, 1L));
                }
            }
        }
    }

    private void sendMessageToNeighbors(List<RowEdge> outEdges, Object message) {
        for (RowEdge rowEdge : outEdges) {
            context.sendMessage(rowEdge.getTargetId(), message);
        }
    }

    @Override
    public void finish(RowVertex vertex) {
        if (vertex.getId().equals(sourceId)) {
            long len = (long) vertex.getValue().getField(0, LongType.INSTANCE);
            long num = (long) vertex.getValue().getField(1, LongType.INSTANCE);
            context.take(ObjectRow.create(vertex.getId(), (double) num / len));
        }
    }

    @Override
    public StructType getOutputType() {
        return new StructType(
            new TableField("id", LongType.INSTANCE, false),
            new TableField("cc", DoubleType.INSTANCE, false)
        );
    }
}

```

然后，可在 DSL 中引入自定义算法，直接调用使用，语法形式如下：
```sql
CREATE Function closeness_centrality AS 'com.antgroup.geaflow.dsl.udf.ClosenessCentrality';

INSERT INTO tbl_result
CALL closeness_centrality(1) YIELD (vid, ccValue)
RETURN vid, ROUND(ccValue, 3)
;
```
示例表示，计算图中 `id = 1`节点的紧密中心度。



## 算法运行
在运行算法之前，要构造算法运行的底图数据。



### 图定义
首先，进行图定义：
```sql
CREATE GRAPH modern (
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
	  weight double
	),
	Edge created (
	  srcId bigint SOURCE ID,
  	targetId bigint DESTINATION ID,
  	weight double
	)
) WITH (
	storeType='rocksdb',
	shardNum = 1
);
```


### 图构建

完成图定义之后，导入点边数据，构造数据底图：
```sql
CREATE TABLE modern_vertex (
  id varchar,
  type varchar,
  name varchar,
  other varchar
) WITH (
  type='file',
  geaflow.dsl.file.path = 'resource:///data/modern_vertex.txt'
);

CREATE TABLE modern_edge (
  srcId bigint,
  targetId bigint,
  type varchar,
  weight double
) WITH (
  type='file',
  geaflow.dsl.file.path = 'resource:///data/modern_edge.txt'
);

INSERT INTO modern.person
SELECT cast(id as bigint), name, cast(other as int) as age
FROM modern_vertex WHERE type = 'person'
;

INSERT INTO modern.software
SELECT cast(id as bigint), name, cast(other as varchar) as lang
FROM modern_vertex WHERE type = 'software'
;

INSERT INTO modern.knows
SELECT srcId, targetId, weight
FROM modern_edge WHERE type = 'knows'
;

INSERT INTO modern.created
SELECT srcId, targetId, weight
FROM modern_edge WHERE type = 'created'
;
```


### 计算输出

最后，在底图数据上完成算法计算和结果输出；
```sql
CREATE TABLE tbl_result (
  vid int,
	ccValue double
) WITH (
	type='file',
	geaflow.dsl.file.path='/tmp/result'
);

CREATE Function closeness_centrality AS 'com.antgroup.geaflow.dsl.udf.ClosenessCentrality';

USE GRAPH modern;

INSERT INTO tbl_result
CALL closeness_centrality(1) YIELD (vid, ccValue)
RETURN vid, ROUND(ccValue, 3)
;
```


### 运行示例

- input
```sql
// vertex
1,person,marko,29
2,person,vadas,27
3,software,lop,java
4,person,josh,32
5,software,ripple,java
6,person,peter,35

// edge
1,3,created,0.4
1,2,knows,0.5
1,4,knows,1.0
4,3,created,0.4
4,5,created,1.0
3,6,created,0.2
```

- output
```sql
// result
1,0.714
```


## 结语

在本篇文章中我们介绍了如何在TuGraph Analytics上实现紧密中心度算法，如果你觉得比较有趣，欢迎关注我们的社区（[https://github.com/TuGraph-family/tugraph-analytics](https://github.com/TuGraph-family/tugraph-analytics)）。开源不易，如果你觉得还不错，可以给我们star支持一下～

------------------------

GeaFlow(品牌名TuGraph-Analytics) 已正式开源，欢迎大家关注！！！

欢迎给我们 Star 哦!

Welcome to give us a Star!

GitHub👉[https://github.com/TuGraph-family/tugraph-analytics](https://github.com/TuGraph-family/tugraph-analytics)

## 微信群
![image](../../../../assets/images/wechat.png)

请点击项目链接下方微信二维码添加微信用户群：[https://github.com/TuGraph-family/tugraph-analytics](https://github.com/TuGraph-family/tugraph-analytics)
