---
layout: post
read_time: true
show_date: true
show_author: true
title: "GitHub上有哪些好项目？GeaFlow图计算快速上手之SSSP算法"
date: 2023-07-21
tags: [蚂蚁集团, 图计算, 单源最短路径, GeaFlow, 开源, github]
category: news
author: TuGraph
description: "本文介绍了实时图计算引擎GeaFlow支持图算法SSSP的基本原理以及在GeaFlow中的实现细节和使用方式。"
---

## 引言
下面这张图是GitHub中约500个开源项目仓库与话题组成的关系网络，密布的连线恐怕没有人能从中找到任何有用的信息。然而GitHub目前总共有3000000+的仓库！

![image](../../../../assets/images/posts/20230721/github1.png)

如何在5分钟内发现有哪些我们感兴趣好项目？

今天我们使用GeaFlow帮助我们实现SSSP(单源最短路径算法)，来试一试盲人摸象！

GeaFlow(品牌名TuGraph-Analytics)是蚂蚁集团开源的分布式实时图计算引擎，目前广泛应用于金融风控、社交网络、知识图谱以及数据应用等场景。

## SSSP(单源最短路径算法)算法介绍
SSSP单源最短路径算法（Single Source Shortest Path）是一种基于图论的算法，用于寻找一个起点到其他所有节点的最短路径。该算法可以应用于多种实际问题，如地图导航、网络拓扑等。

在GitHub开源项目仓库与话题组成的关系网络中，从仓库到话题再到仓库的关系边可以支持SSSP算法的运行。

![image](../../../../assets/images/posts/20230721/github2.png)

如图，在关系网络局部，从起点出发，通过箭头的个数可以标记话题/仓库到源点的距离。例如仓库FiraCode与仓库Font-Awesome的距离为2，通过2个箭头可到达，它们也是互相距离最近的关联仓库。

简单来说，标记出我们感兴趣的仓库，那些与我们感兴趣仓库距离最近的仓库就是推荐的好仓库。或者更进一步，STAR数更多的近距离仓库更值得推荐。

## GeaFlow实现SSSP
要运行SSSP算法，我们可以指定使用的图，直接在图查询里调用图算法，语法形式如下：

```sql
USE GRAPH github_repo_topic
INSERT INTO tbl_result
CALL sssp('source_vertex') YIELD (repoName, distance)
RETURN repoName, distance;
```

这段代码在图github_repo_topic上运行，将source_vertex作为算法起点，输出所有其他点的距离。如果无需这么多结果，可以在RETURN中加上WHERE条件过滤，一切和SQL语句一样！

如果需要定制一个图算法，我们可以实现AlgorithmUserFunction接口。GeaFlow内置了多种图算法的通用实现，这些算法无需单独定制，例如SSSP算法的参考实现如下：

```java
@Description(name = "sssp", description = "built-in udga Single Source Shortest Path")
public class SSSP implements AlgorithmUserFunction<Object, Long> {

    private AlgorithmRuntimeContext<Object, Long> context;
    
    @Override
    public void init(AlgorithmRuntimeContext<Object, Long> context, Object[] parameters) {
        //初始化算法上下文
        this.context = context;
    }

    @Override
    public void process(RowVertex vertex, Iterator<Long> messages) {
        long currentDistance;
        //初始化所有点距离初始值
        if (context.getCurrentIterationId() == 1L) {
            //初始化所有点距离初始值
        } else if (context.getCurrentIterationId() <= $maxIteration) {
            //计算最短距离
        } else {
            //返回结果
        }
        //更新距离值
        context.updateVertexValue(ObjectRow.create($currentDistance));
        //向邻居发送消息
        context.sendMessage(vertex.getId(), $currentDistance);
        long scatterDistance = $currentDistance == Long.MAX_VALUE ? Long.MAX_VALUE : currentDistance + 1;
        for (RowEdge edge : context.loadEdges(EdgeDirection.OUT)) {
            context.sendMessage(edge.getTargetId(), scatterDistance);
        }
    }

    @Override
    public StructType getOutputType() {
        //算法返回值数据类型
        return new StructType(
            new TableField("id", StringType.INSTANCE, false), 
            new TableField("distance", LongType.INSTANCE, false)
        );
    }
}
```

图查询以提交作业的形式完成，作业可以运行在本地或K8S集群中，GeaFlow提供控制台管理和回溯这些图研发作业。

![image](../../../../assets/images/posts/20230721/github3.png)

## 在GitHub关系图上盲人摸象

话不多说，我们找到GitHub上目前星星数最多的项目，计算与它距离为2（即具有共同话题）的项目都有哪些？

目前星星最多的项目是freeCodeCamp，这里数据[GitHub Public Repository Metadata](https://www.kaggle.com/datasets/pelmers/github-repository-metadata-with-5-stars)截止2023年5月。

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

短短时间我们就拿到了计算结果，来看看GeaFlow都给我淘到了哪些好项目吧。这里不按星星数排序，随机呈现10条记录。
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

## 总结
本文介绍了实时图计算引擎GeaFlow支持图算法SSSP的基本原理以及在GeaFlow中的实现细节，并展示其在GitHub数据集上的一个应用。



GeaFlow(品牌名TuGraph-Analytics) 已正式开源，欢迎大家关注！！！

欢迎给我们 Star 哦!

Welcome to give us a Star!

GitHub👉[https://github.com/TuGraph-family/tugraph-analytics](https://github.com/TuGraph-family/tugraph-analytics)

## 微信群
![image](../../../../assets/images/wechat.png)

请点击项目链接下方微信二维码添加微信用户群：[https://github.com/TuGraph-family/tugraph-analytics](https://github.com/TuGraph-family/tugraph-analytics)




