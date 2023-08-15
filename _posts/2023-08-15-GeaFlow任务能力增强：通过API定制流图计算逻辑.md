---
layout: post
read_time: true
show_date: true
show_author: true
title: "GeaFlow任务能力增强：通过API定制流图计算逻辑"
date: 2023-08-15
tags: [图计算, 高阶API, TuGraph, GeaFlow, Java]
category: opinion
author: TuGraph
description: "GeaFlow API是对高阶用户提供的开发接口，用户可以直接通过编写java代码来编写计算作业，相比于DSL，API的方式开发更加灵活，也能实现更丰富的功能和更复杂的计算逻辑。"
---

# GeaFlow API介绍
GeaFlow API是对高阶用户提供的开发接口，用户可以直接通过编写java代码来编写计算作业，相比于DSL，API的方式开发更加灵活，也能实现更丰富的功能和更复杂的计算逻辑。
在GeaFlow中，API支持Graph API和Stream API两种类型：

- Graph API：Graph是GeaFlow框架的一等公民，当前GeaFlow框架提供了一套基于GraphView的图计算编程接口，包含图构建、图计算及遍历。在GeaFlow中支持Static Graph和Dynamic Graph两种类型。
  - Static Graph API：静态图计算API，基于该类API可以进行全量的图计算或图遍历。
  - Dynamic Graph API：动态图计算API，GeaFlow中GraphView是动态图的数据抽象，基于GraphView
    之上，可以进行动态图计算或图遍历。同时支持对Graphview生成Snapshot快照，基于Snapshot可以提供和Static Graph API一样的接口能力。

![image](../../../../assets/images/posts/20230815/001.jpeg)

- Stream API：GeaFlow提供了一套通用计算的编程接口，包括source构建、流批计算及sink输出。在GeaFlow中支持Batch和Stream两种类型。
  - Batch API：批计算API，基于该类API可以进行批量计算。
  - Stream API：流计算API，GeaFlow中StreamView是动态流的数据抽象，基于StreamView之上，可以进行流计算。

更多API的介绍可参考 [https://github.com/TuGraph-family/tugraph-analytics/blob/master/docs/docs-cn/application-development/api/overview.md](https://github.com/TuGraph-family/tugraph-analytics/blob/master/docs/docs-cn/application-development/api/overview.md)
# PageRank算法示例
本例子是从文件中读取点边进行构图，执行pageRank算法后，将每个点的pageRank值进行打印。
其中，用户需要实现AbstractVcFunc，在compute方法中进行每一轮迭代的计算逻辑。
在本例子中，只计算了两轮迭代的结果。在第一轮中，每个点都会向邻居点发送当前点的value值，而在第二轮中，每个点收到邻居点发送的消息，将其value值进行累加，并更新为自己的value值，即为最后的PageRank值。
```java
public class PageRank {

    private static final Logger LOGGER = LoggerFactory.getLogger(PageRank.class);

    public static final String RESULT_FILE_PATH = "./target/tmp/data/result/pagerank";

    private static final double alpha = 0.85;

    public static void main(String[] args) {
        Environment environment = EnvironmentUtil.loadEnvironment(args);
        IPipelineResult result = PageRank.submit(environment);
        PipelineResultCollect.get(result);
        environment.shutdown();
    }

    public static IPipelineResult submit(Environment environment) {
        Pipeline pipeline = PipelineFactory.buildPipeline(environment);
        Configuration envConfig = environment.getEnvironmentContext().getConfig();
        envConfig.put(FileSink.OUTPUT_DIR, RESULT_FILE_PATH);
        ResultValidator.cleanResult(RESULT_FILE_PATH);

        pipeline.submit((PipelineTask) pipelineTaskCxt -> {
            Configuration conf = pipelineTaskCxt.getConfig();
            PWindowSource<IVertex<Integer, Double>> prVertices =
                pipelineTaskCxt.buildSource(new FileSource<>("email_vertex",
                        line -> {
                            String[] fields = line.split(",");
                            IVertex<Integer, Double> vertex = new ValueVertex<>(
                                Integer.valueOf(fields[0]), Double.valueOf(fields[1]));
                            return Collections.singletonList(vertex);
                        }), AllWindow.getInstance())
                    .withParallelism(conf.getInteger(ExampleConfigKeys.SOURCE_PARALLELISM));

            PWindowSource<IEdge<Integer, Integer>> prEdges = pipelineTaskCxt.buildSource(new FileSource<>("email_edge",
                    line -> {
                        String[] fields = line.split(",");
                        IEdge<Integer, Integer> edge = new ValueEdge<>(Integer.valueOf(fields[0]), Integer.valueOf(fields[1]), 1);
                        return Collections.singletonList(edge);
                    }), AllWindow.getInstance())
                .withParallelism(conf.getInteger(ExampleConfigKeys.SOURCE_PARALLELISM));

            int iterationParallelism = conf.getInteger(ExampleConfigKeys.ITERATOR_PARALLELISM);
            GraphViewDesc graphViewDesc = GraphViewBuilder
                .createGraphView(GraphViewBuilder.DEFAULT_GRAPH)
                .withShardNum(2)
                .withBackend(BackendType.Memory)
                .build();
            PGraphWindow<Integer, Double, Integer> graphWindow =
                pipelineTaskCxt.buildWindowStreamGraph(prVertices, prEdges, graphViewDesc);

            SinkFunction<IVertex<Integer, Double>> sink = ExampleSinkFunctionFactory.getSinkFunction(conf);
            graphWindow.compute(new PRAlgorithms(10))
                .compute(iterationParallelism)
                .getVertices()
                .sink(v -> {
                    LOGGER.info("result {}", v);
                })
                .withParallelism(conf.getInteger(ExampleConfigKeys.SINK_PARALLELISM));
        });

        return pipeline.execute();
    }

    public static class PRAlgorithms extends VertexCentricCompute<Integer, Double, Integer, Double> {

        public PRAlgorithms(long iterations) {
            super(iterations);
        }

        @Override
        public VertexCentricComputeFunction<Integer, Double, Integer, Double> getComputeFunction() {
            return new PRVertexCentricComputeFunction();
        }

        @Override
        public VertexCentricCombineFunction<Double> getCombineFunction() {
            return null;
        }

    }

    public static class PRVertexCentricComputeFunction extends AbstractVcFunc<Integer, Double, Integer, Double> {

        @Override
        public void compute(Integer vertexId,
                            Iterator<Double> messageIterator) {
            IVertex<Integer, Double> vertex = this.context.vertex().get();
            List<IEdge<Integer, Integer>> outEdges = context.edges().getOutEdges();
            if (this.context.getIterationId() == 1) {
                if (!outEdges.isEmpty()) {
                    this.context.sendMessageToNeighbors(vertex.getValue() / outEdges.size());
                }

            } else {
                double sum = 0;
                while (messageIterator.hasNext()) {
                    double value = messageIterator.next();
                    sum += value;
                }
                double pr = sum * alpha + (1 - alpha);
                this.context.setNewVertexValue(pr);

                if (!outEdges.isEmpty()) {
                    this.context.sendMessageToNeighbors(pr / outEdges.size());
                }
            }
        }

    }
}

```
# 提交API作业
(以容器模式，PageRank算法示例)
## 算法打包
在新的项目中新建一个PageRank的demo，pom中引入geaflow依赖
```java
<dependency>
	<groupId>com.antgroup.tugraph</groupId>
	<artifactId>geaflow-assembly</artifactId>
	<version>0.2-SNAPSHOT</version>
</dependency>
```

新建PageRank类，编写上述相关[代码](#xBvDN)。
在项目resources路径下，创建测试数据文件email_vertex和email_edge，代码中会从resources://资源路径读取数据进行构图。
```java
 PWindowSource<IVertex<Integer, Double>> prVertices =
                pipelineTaskCxt.buildSource(new FileSource<>("email_vertex",
                        line -> {
                            String[] fields = line.split(",");
                            IVertex<Integer, Double> vertex = new ValueVertex<>(
                                Integer.valueOf(fields[0]), Double.valueOf(fields[1]));
                            return Collections.singletonList(vertex);
                        }), AllWindow.getInstance())
                    .withParallelism(conf.getInteger(ExampleConfigKeys.SOURCE_PARALLELISM));

   PWindowSource<IEdge<Integer, Integer>> prEdges = pipelineTaskCxt.buildSource(new FileSource<>("email_edge",
                    line -> {
                        String[] fields = line.split(",");
                        IEdge<Integer, Integer> edge = new ValueEdge<>(Integer.valueOf(fields[0]), Integer.valueOf(fields[1]), 1);
                        return Collections.singletonList(edge);
                    }), AllWindow.getInstance())
                .withParallelism(conf.getInteger(ExampleConfigKeys.SOURCE_PARALLELISM));
```
email_vertex
```
0,1
1,1
2,1
3,1
4,1
5,1
6,1
7,1
8,1
9,1
```
email_edge
```
4,3
0,1
2,3
4,6
2,4
6,8
0,2
4,8
0,5
0,7
0,8
9,0
7,0
7,1
7,2
9,5
3,0
7,4
5,3
7,5
1,0
5,4
9,8
3,4
7,9
3,7
3,8
1,6
8,0
6,0
6,2
8,5
4,2
```
maven打包，在target目录获取算法的jar包
```powershell
mvn clean install
```
## 新增HLA图任务
在GeaFlow Console中新增图任务，任务类型选择“HLA”， 并上传jar包（或者选择已存在的jar包），其中**entryClass为算法主函数所在的类**。 点击“提交”，创建任务。

![image](../../../../assets/images/posts/20230815/002.png)

##  提交作业

![image](../../../../assets/images/posts/20230815/003.png)

点击"发布"，可进入作业详情界面，点击“提交”即可提交作业。

![image](../../../../assets/images/posts/20230815/004.png)

## 查看运行结果
进入容器 /tmp/logs/task/ 目录下，查看对应作业的日志，可看到日志中打印了最终计算得到的每个点的pageRank值。
```abap
2023-08-01 16:51:38 INFO  PageRank:107 - result ValueVertex(vertexId:0, value:1.5718675107490019)
2023-08-01 16:51:38 INFO  PageRank:107 - result ValueVertex(vertexId:1, value:0.5176947080197076)
2023-08-01 16:51:38 INFO  PageRank:107 - result ValueVertex(vertexId:2, value:1.0201253300467092)
2023-08-01 16:51:38 INFO  PageRank:107 - result ValueVertex(vertexId:3, value:1.3753756869824914)
2023-08-01 16:51:38 INFO  PageRank:107 - result ValueVertex(vertexId:4, value:1.4583114077692536)
2023-08-01 16:51:38 INFO  PageRank:107 - result ValueVertex(vertexId:5, value:1.1341668910561529)
2023-08-01 16:51:38 INFO  PageRank:107 - result ValueVertex(vertexId:6, value:0.6798184364673463)
2023-08-01 16:51:38 INFO  PageRank:107 - result ValueVertex(vertexId:7, value:0.70935427506243)
2023-08-01 16:51:38 INFO  PageRank:107 - result ValueVertex(vertexId:8, value:1.2827529511906106)
2023-08-01 16:51:38 INFO  PageRank:107 - result ValueVertex(vertexId:9, value:0.2505328026562969)
```

可在作业详情中查看运行详情，

![image](../../../../assets/images/posts/20230815/005.png)

**至此，我们就成功使用Geaflow实现并运行API任务了！是不是超简单！快来试一试吧！**


GeaFlow(品牌名TuGraph-Analytics) 已正式开源，欢迎大家关注！！！

欢迎给我们 Star 哦!

Welcome to give us a Star!

GitHub👉[https://github.com/TuGraph-family/tugraph-analytics](https://github.com/TuGraph-family/tugraph-analytics)

## 微信群
![image](../../../../assets/images/wechat.png)

请点击项目链接下方微信二维码添加微信用户群：[https://github.com/TuGraph-family/tugraph-analytics](https://github.com/TuGraph-family/tugraph-analytics)




