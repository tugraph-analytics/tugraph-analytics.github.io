---
layout: post
read_time: true
show_date: true
show_author: true
title: "一张图读懂TuGraph Analytics开源技术架构"
date: 2023-08-21
tags: [架构, 分布式计算, SQL, TuGraph-Analytics, 开源, GQL]
category: opinion
author: 范志东
description: "TuGraph Analytics（内部项目名GeaFlow）是蚂蚁集团开源的分布式实时图计算引擎，即流式图计算。通过SQL+GQL融合分析语言对表模型和图模型进行统一处理，实现了流、批、图一体化计算，并支持了Exactly Once语义、高可用以及一站式图研发平台等生产化能力。"
---

作者：范志东

**TuGraph Analytics（内部项目名GeaFlow）**是蚂蚁集团开源的分布式实时图计算引擎，即流式图计算。通过SQL+GQL融合分析语言对表模型和图模型进行统一处理，实现了流、批、图一体化计算，并支持了Exactly Once语义、高可用以及一站式图研发平台等生产化能力。

开源项目代码目前托管在GitHub，欢迎业界同仁、大数据/图计算技术爱好者关注我们的项目并参与共建。

**项目地址：[https://github.com/TuGraph-family/tugraph-analytics](https://github.com/TuGraph-family/tugraph-analytics)**

**GeaFlow论文【SIGMOD 2023】：[GeaFlow: A Graph Extended and Accelerated Dataflow System](https://dl.acm.org/doi/abs/10.1145/3589771)**

## 概览

本文希望通过一张图描述清楚TuGraph Analytics的整体架构脉络和关键设计思路，以帮助大家快速对TuGraph Analytics项目的轮廓有个整体的认识。闲言少叙，直接上图。

![TuGraph Analytics开源技术架构](../../../../assets/images/posts/20230821/tu1.png)

TuGraph Analytics开源技术架构一共分为五个部分：

* **DSL层**：即语言层。TuGraph Analytics设计了SQL+GQL的融合分析语言，支持对表模型和图模型统一处理。
* **Framework层**：即框架层。TuGraph Analytics设计了面向Graph和Stream的两套API支持流、批、图融合计算，并实现了基于Cycle的统一分布式调度模型。
* **State层**：即存储层。TuGraph Analytics设计了面向Graph和KV的两套API支持表数据和图数据的混合存储，整体采用了Sharing Nothing的设计，并支持将数据持久化到远程存储。
* **Console平台**：TuGraph Analytics提供了一站式图研发平台，实现了图数据的建模、加工、分析能力，并提供了图作业的运维管控支持。
* **执行环境**：TuGraph Analytics可以运行在多种异构执行环境，如K8S、Ray以及本地模式。

## DSL层

DSL层是一个典型的编译器技术架构，即语法分析、语义分析、中间代码生成(IR)、代码优化、目标代码生成（OBJ）的流程。

![DSL层架构](../../../../assets/images/posts/20230821/tu2.png)

* **语言设计**：TuGraph Analytics设计了SQL+GQL的融合语法，解决了图+表一体化分析的诉求。具体语法设计可以参考文章：[DSL语法文档](https://github.com/TuGraph-family/tugraph-analytics/blob/master/docs/docs-en/application-development/dsl/overview.md)
* **语法分析**：通过扩展Calcite的SqlNode和SqlOperator，实现SQL+GQL的语法解析器，生成统一的语法树信息。
* **语义分析**：通过扩展Calcite的Scope和Namespace，实现自定义Validator，对语法树进行约束语义检查。
* **中间代码生成**：通过扩展Calcite的RelNode，实现图上的Logical RelNode，用于GQL语法的中间表示。
* **代码优化**：优化器实现了大量的优化规则（RBO）用于提升执行性能，未来也会引入CBO。
* **目标代码生成**：代码生成器Converter负责将Logical RelNode转换为Physical RelNode，即目标代码。Physical RelNode可以直接翻译为Graph/Table上的API调用。
* **自定义函数**: TuGraph Analytics提供了大量的内置系统函数，用户也可以根据需要注册自定义函数。
* **自定义插件**: TuGraph Analytics允许用户扩展自己的Connector类型，以支持不同的数据源和数据格式。

## Framework层

Framework层设计与Flink/Spark等同类大数据计算引擎有一定的相似性，即提供了类FlumeJava（[FlumeJava: Easy, Efficient Data-Parallel Pipelines](https://pages.cs.wisc.edu/~akella/CS838/F12/838-CloudPapers/FlumeJava.pdf)）的统一高阶API（简称HLA），用户调用高阶API的过程会被转换为逻辑执行计划，逻辑执行计划执行一定的优化（如ChainCombine、UnionPushUp等）后，被转换为物理执行计划，物理执行计划会被调度器分发到分布式Worker上执行，最终Worker会回调用户传递的高阶API函数逻辑，实现整个分布式计算链路的执行。

![Framework层架构](../../../../assets/images/posts/20230821/tu3.png)

* **高阶API**：TuGraph Analytics通过Environment接口适配异构的分布式执行环境（K8S、Ray、Local），使用Pipeline封装了用户的数据处理流程，使用Window抽象统一了流处理（无界Window）和批处理（有界Window）。Graph接口提供了静态图和动态图（流图）上的计算API，如append/snapshot/compute/traversal等，Stream接口提供了统一流批处理API，如map/reduce/join/keyBy等。
* **逻辑执行计划**：逻辑执行计划信息统一封装在PipelineGraph对象内，将高阶API对应的算子（Operator）组织在DAG中，算子一共分为5大类：SourceOperator对应数据源加载、OneInputOperator/TwoInputOperator对应传统的数据处理、IteratorOperator对应静态/动态图计算。DAG中的点（PipelineVertex）记录了算子（Operator）的关键信息，如类型、并发度、算子函数等信息，边（PipelineEdge）则记录了数据shuffle的关键信息，如Partition规则（forward/broadcast/key等）、编解码器等。
* **物理执行计划**：物理执行计划信息统一封装在ExecutionGraph对象内，并支持二级嵌套结构，以尽可能将可以流水线执行的子图（ExecutionVertexGroup）结构统一调度。图中示例的物理执行计划DAG被划分为三部分子图结构分别执行。
* **调度器**：TuGraph Analytics设计了基于Cycle的调度器（CycleScheduler）实现对流、批、图的统一调度，调度过程通过事件驱动模型触发。物理执行计划中的每部分子图都会被转换为一个ExecutionCycle对象，调度器会向Cycle的头结点（Head）发送Event，并接收Cycle尾结点（Tail）的发回的Event，形成一个完整的调度闭环。对于流处理，每一轮Cycle调度会完成一个Window的数据的处理，并会一直不停地执行下去。对于批处理，整个Cycle调度仅执行一轮。对于图处理，每一轮Cycle调度会完成一次图计算迭代。
* **运行时组件**：TuGraph Analytics运行时会拉起Client、Master、Driver、Container组件。当Client提交Pipeline给Driver后，会触发执行计划构建、分配Task（ResourceManagement提供资源）和调度。每个Container内可以运行多个Worker组件，不同Worker组件之间通过Shuffle模块交换数据，所有的Worker都需要定期向Master上报心跳（HeartbeatManagement），并向时序数据库上报运行时指标信息。另外TuGraph Analytics运行时也提供了故障容忍机制（FailOver），以便在异常/中断后能继续执行。

## State层

State层设计相比于传统的大数据计算引擎，除了提供面向表数据的KV存储抽象，也支持了面向图数据的Graph存储抽象，以更好地支持面向图模型的IO性能优化。

![State层架构](../../../../assets/images/posts/20230821/tu4.png)

* **State API**：提供了面向KV存储API，如get/put/delete等。以及面向图存储的API，如V/E/VE，以及点/边的add/update/delete等。
* **State执行层**：通过KeyGroup的设计实现数据的Sharding和扩缩容能力，Accessor提供了面向不同读写策略和数据模型的IO抽象，StateOperator抽象了存储层SPI，如finish（刷盘）、archive（Checkpoint）、compact（压缩）、recover（恢复）等。另外，State提供了多种PushDown优化以加速IO访问效率。通过自定义内存管理和面向属性的二级索引也会提供大量的存储访问优化手段。
* **Store层**：TuGraph Analytics支持了多种存储系统类型，并通过StoreContext封装了Schema、序列化器，以及数据版本信息。
* **持久化层**：State的数据支持持久化到远程存储系统，如HDFS、OSS、S3等。

## Console平台

Console平台提供了一站式图研发、运维的平台能力，同时为引擎运行时提供元数据（Catalog）服务。

![Console平台架构](../../../../assets/images/posts/20230821/tu5.png)

* **标准化API**：平台提供了标准化的RESTful API和认证机制，同时支持了页面端和应用端的统一API服务能力。
* **任务研发**：平台支持“关系-实体-属性”的图数据建模。基于字段映射配置，可以定义图数据传输任务，包括数据集成（Import）和数据分发（Export）。基于图表模型的图数据加工任务支持多样化的计算场景，如Traversal、Compute、Mining等。基于数据加速器的图数据服务，提供了多协议的实时分析能力，支持BI、可视化分析工具的接入集成。
* **构建提交**：平台通过任务和作业的独立抽象，实现研发态与运维态的分离。任务开发完成后执行发布动作，会自动触发构建流水线（Release Builder），生成发布版本。任务提交器（Task Submitter）负责将发布版本的内容提交到执行环境，生成计算作业。
* **作业运维**：作业属于任务的运行态，平台提供了作业的操纵（启停、重置）、监控（指标、告警、审计）、调优（诊断、伸缩、调参）、调度等运维能力。作业的运行时资源会由资源池统一分配和管理。
* **元数据服务**：平台同时承载了引擎运行时的元数据服务能力，以实现研发与运维的自动化。元数据以实例维度进行隔离，实例内的研发资源可以根据名字直接访问，如点、边、图、表、视图、函数等。
* **系统管理**：平台提供了多租户隔离机制、细粒度用户权限控制，以及系统资源的管理能力。

## 执行环境

TuGraph Analytics支持多种异构环境执行，以常见的K8S部署环境为例，其物理部署架构如下：

![TuGraph Analytics K8S部署架构](../../../../assets/images/posts/20230821/tu6.png)

在TuGraph Analytics作业的全生命周期过程中，涉及的关键数据流程有：

* **研发阶段**：Console平台提供了实例下所有的研发资源的管理，用户可以在创建任务前，提前准备所需的研发资源信息，并存储在Catalog。
* **构建阶段**：任务创建完成后，通过发布动作触发构建流水线，用户的JAR包、任务的ZIP包等会上传到RemoteFileStore。
* **提交阶段**：作业提交时，Console会根据作业的参数配置、运行时环境信息，以及远程文件地址等创建KubernetesJobClient，既而会拉起Client Pod，Client会拉起Master Pod，Master会拉起Container Pods和Driver Pod。所有的Pod拉起后，Client会把作业的Pipeline发送给Driver执行，Driver最终通过Cycle调度的Events与Containers交互。所有的Pod启动时都会从RemoteFileStore下载版本JAR包、用户JAR包、作业ZIP包等信息。Driver对DSL代码编译时，也需要通过Console提供的Catalog API操作Schema信息。
* **运行阶段**：作业运行时，各个组件会上报不同的数据和信息。Master会上报作业的心跳汇总信息，Driver会上报作业的Pipeline/Cycle指标以及错误信息，Container会上报作业的Offset、指标定义以及错误信息等。RuntimeMetaStore存储作业的Pipeline/Cycle指标、Offset、心跳汇总、错误等信息。HAMetaStore存储各个运行组件的地址信息。DataStore存储State数据和作业FailOver时所需的元数据信息。MetricStore存储运行时指标信息。
* **监控阶段**：Console会主要查询RuntimeMetaStore和MetricStore存储的信息用于作业的运行时监控。
* **清理阶段**：作业重置/删除时，Console会对作业的RuntimeMeta、HAMeta以及部分Data做清理操作。

## 总结

希望通过以上的介绍，可以让大家对TuGraph Analytics开源技术架构有个比较清晰的了解，我们非常欢迎开源社区的技术爱好者参与到项目的建设中来。

如果您对TuGraph Analytics项目比较感兴趣，欢迎动动手指扫码直达GitHub仓库，为我们的项目加一颗Star。【网络不畅可以尝试使用VPN访问】

GeaFlow(品牌名TuGraph-Analytics) 已正式开源，欢迎大家关注！！！

欢迎给我们 Star 哦!

Welcome to give us a Star!

GitHub👉[https://github.com/TuGraph-family/tugraph-analytics](https://github.com/TuGraph-family/tugraph-analytics)

## 微信群
![image](../../../../assets/images/wechat.png)

请点击项目链接下方微信二维码添加微信用户群：[https://github.com/TuGraph-family/tugraph-analytics](https://github.com/TuGraph-family/tugraph-analytics)
