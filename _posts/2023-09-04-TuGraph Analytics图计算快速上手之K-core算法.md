---
layout: post
read_time: true
show_date: true
show_author: true
title: "TuGraph Analytics图计算快速上手之K-core算法"
date: 2023-09-04
tags: [图计算, 算法, K-core, TuGraph-Analytics, 开源, GQL]
category: opinion
author: 郑光杰
description: "在本篇文章中我们介绍了如何在TuGraph Analytics上实现K-Core算法。"
---

作者：郑光杰

## 引言
K-Core算法是一种用来在图中找出符合指定核心度的紧密关联的子图结构，在K-Core的结果子图中，每个顶点至少具有k的度数，且所有顶点都至少与该子图中的 k 个其他节点相连。K-Core通常用来对一个图进行子图划分，通过去除不重要的顶点，将符合逾期的子图暴露出来进行进一步分析。K-Core图算法常用来识别和提取图中的紧密连通群组，因具有较低的时间复杂度（线性）及较好的直观可解释性，广泛应用于金融风控、社交网络和生物学等研究领域。
## K-Core算法介绍
一张图的 K-Core子图是指从图中反复去掉度（不考虑自环边）小于 k 的节点之后得到的子图。该计算过程是一个反复迭代剪枝的过程，在某一轮剪枝之前度大于等于 k 的节点，可能会在该轮剪枝后变为度小于 k。比如3-core子图的切分过程如图1所示：

![image.png](../../../../assets/images/posts/20230904/tu0.png)

3-core子图切分过程
## TuGraph-Analytics实现K-Core算法

要运行K-Core算法，我们可以指定使用的图，直接在图查询里调用K-Core算法，语法形式如下：
```sql
INSERT INTO tbl_result
CALL kcore(3) YIELD (id, value)
RETURN id, value;
```
运行该语法之后，就可以从图中查询到k=3的子图的id以及该id的邻居数。
TuGraph-Analytics 已经内置了许多算法，如果想要自定义算法，可以基于AlgorithmUserFunction接口实现，比如自定义k-core 算法实现如下：
```java
package com.tugraph.demo;

import com.antgroup.geaflow.common.type.primitive.IntegerType;
import com.antgroup.geaflow.dsl.common.algo.AlgorithmRuntimeContext;
import com.antgroup.geaflow.dsl.common.algo.AlgorithmUserFunction;
import com.antgroup.geaflow.dsl.common.data.RowEdge;
import com.antgroup.geaflow.dsl.common.data.RowVertex;
import com.antgroup.geaflow.dsl.common.data.impl.ObjectRow;
import com.antgroup.geaflow.dsl.common.function.Description;
import com.antgroup.geaflow.dsl.common.types.StructType;
import com.antgroup.geaflow.dsl.common.types.TableField;
import com.antgroup.geaflow.model.graph.edge.EdgeDirection;
import java.util.Iterator;
import java.util.List;

@Description(name = "kcore", description = "built-in udga for KCore")
public class KCore implements AlgorithmUserFunction<Object, Integer> {

    private AlgorithmRuntimeContext<Object, Integer> context;
    private int k = 1;

    @Override
    public void init(AlgorithmRuntimeContext<Object, Integer> context, Object[] params) {
        this.context = context;
        if (params.length > 1) {
            throw new IllegalArgumentException(
                "Only support zero or more arguments, false arguments "
                    + "usage: func([alpha, [convergence, [max_iteration]]])");
        }
        // 设置k值，默认k=1
        if (params.length > 0) {
            k = Integer.parseInt(String.valueOf(params[0]));
        }
    }

    @Override
    public void process(RowVertex vertex, Iterator<Integer> messages) {

        boolean isFinish = false;
        //第一轮迭代将所有顶点初始化，目标点的value值初始化为-1，并向邻点发送消息
        if (this.context.getCurrentIterationId() == 1) {
            this.context.updateVertexValue(ObjectRow.create(-1));
        } else {
            // v = 0,则表示需要删除
            int currentV = (int) vertex.getValue().getField(0, IntegerType.INSTANCE);
            if (currentV == 0) {
                return;
            }

            // 计算点的输入消息数
            int sum = 0;
            while (messages.hasNext()) {
                sum += messages.next();
            }
            // 如果点接收的消息数小于k的则需要删除
            if (sum < k) {
                isFinish = true;
                sum = 0;
            }
            // 更新当前点的值为接收消息数
            context.updateVertexValue(ObjectRow.create(sum));
        }

        if (isFinish) {
            return;
        }

        // 向点的邻居发送消息
        List<RowEdge> outEdges = this.context.loadEdges(EdgeDirection.OUT);
        for (RowEdge rowEdge : outEdges) {
            context.sendMessage(rowEdge.getTargetId(), 1);
        }

        List<RowEdge> inEdges = this.context.loadEdges(EdgeDirection.IN);
        for (RowEdge rowEdge : inEdges) {
            context.sendMessage(rowEdge.getTargetId(), 1);
        }
        // 向本点送消息，防止该点因没有消息不会触发下次迭代
        context.sendMessage(vertex.getId(), 0);
    }

    @Override
    public StructType getOutputType() {
        return new StructType(
            new TableField("id", IntegerType.INSTANCE, false),
            new TableField("v", IntegerType.INSTANCE, false)
        );
    }
}

```

## TuGraph-Analytics运行K-Core算法
### 图定义
如果想要在dsl中运行k-core算法，我们可以第一步先进行图定义，比如：
```sql
CREATE GRAPH IF NOT EXISTS g (
  Vertex v (
    vid int ID,
    value int
  ),
  Edge e (
    srcId int SOURCE ID,
    targetId int DESTINATION ID
  )
) WITH (
  storeType='rocksdb',
  shardCount = 1
);
```
### 图构建
有了图定义之后，我们就可以往这个图中导入点边数据，将这个图构建起来。
```sql
CREATE TABLE IF NOT EXISTS v_source (
    v_id int,
    v_value int
) WITH (
  type='file',
  //vertex文件中保存了点的信息，文件放在与KCore类目录下的resources目录下，此处可以换成其他数据源
  geaflow.dsl.file.path = 'resource:///input/vertex'
);


CREATE TABLE IF NOT EXISTS e_source (
    src_id int,
    dst_id int
) WITH (
  type='file',
    //edge文件中保存了边的信息，文件放在与KCore类目录下的resources目录下，此处可以换成其他数据源
  geaflow.dsl.file.path = 'resource:///input/edge'
);

USE GRAPH g;
INSERT INTO g.v(vid, value)
SELECT
v_id, v_value
FROM v_source;

INSERT INTO g.e(srcId, targetId)
SELECT
 src_id, dst_id
FROM e_source;
```
### 图分析与输出
当图构建之后，我们就可以在图数据基础上进行分析查询和结果输出了。
```sql
//定义结果表
CREATE TABLE IF NOT EXISTS tbl_result (
  v_id int,
  value int
) WITH (
  type='file',
   geaflow.dsl.file.path = '/tmp/result'
);

//注册kcore函数
CREATE Function kcore AS 'com.tugraph.demo.KCore';

USE GRAPH g;
INSERT INTO tbl_result(v_id, value)
//调用kcore函数，并返回结果
CALL kcore(3) YIELD (vid, value)
RETURN vid, value
;
```
### 运行示例
基于以上定义的dsl，我们以图1的数据作为输入，来计算一下图1的3-core子图。
#### 输入
```latex
//vertex文件内容:
1,1
2,1
3,1
4,1
5,1
6,1
7,1
8,1
9,1

//edge文件内容:
1,3
2,3
3,4
3,9
3,5
4,9
4,5
5,9
5,6
5,7
6,7
6,8
7,8
```
#### 输出
```latex
3,3
4,3
5,3
9,3
```
## 总结
在本篇文章中我们介绍了如何在TuGraph Analytics上实现K-Core算法，如果你觉得比较有趣，欢迎关注我们的社区（[https://github.com/TuGraph-family/tugraph-analytics](https://github.com/TuGraph-family/tugraph-analytics)）。开源不易，如果你觉得还不错，可以给我们star支持一下～

------------------------

GeaFlow(品牌名TuGraph-Analytics) 已正式开源，欢迎大家关注！！！

欢迎给我们 Star 哦!

Welcome to give us a Star!

GitHub👉[https://github.com/TuGraph-family/tugraph-analytics](https://github.com/TuGraph-family/tugraph-analytics)

## 微信群
![image](../../../../assets/images/wechat.png)

请点击项目链接下方微信二维码添加微信用户群：[https://github.com/TuGraph-family/tugraph-analytics](https://github.com/TuGraph-family/tugraph-analytics)
