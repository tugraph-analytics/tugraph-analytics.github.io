---
layout: post
read_time: true
show_date: true
show_author: true
title: "TuGraph Analytics云原生部署：基于K8S Operator的轻量级作业启动方案"
date: 2023-11-30
tags: [图计算, 大数据, K8S, 轻量级, Operator, kubernetes]
category: opinion
author: 丁一
description: "TuGraph Analytics云原生部署：基于K8S Operator的轻量级作业启动方案"
---
作者：丁一

### 背景
TuGraph Analytics作业可以通过Console提交部署到K8S集群，但Console是一个独立的Web系统，部署形态上相对较重。在平台工具系统接入或大数据生态集成场景中，需要更轻量级的快速接入TuGraph Analytics的方案。
我们新增了模块geaflow-kubernetes-operator，可以通过更轻量级的YAML文件配置方式，对TuGraph Analytics作业进行描述配置。同时更方便地监控和管理集群下的所有TuGraph Analytics作业，并通过CR(Custom Resource)的创建/修改/删除来管理作业的生命周期和元信息，可以实现只通过kubectl命令实现任务操纵。我们也提供了一个实时dashboard页面，可以方便地白屏化查看所有作业状态和信息。

### 部署K8S Operator
TuGraph Analytics提供了geaflow-kubernetes-operator模块，可通过Helm命令一键部署到K8S。部署完成中，会向K8S集群注册一个名为geaflowjob的自定义资源。（相对于K8S内置pod、service、deployment等系统资源而言）
安装完成后，我们只需要编写一个CR的YAML配置文件提交给K8S，就可以自动拉起作业了。

* 执行以下命令构建Operator镜像，项目代码构建要求JDK11版本，因此需要单独切换JDK版本编译构建。
```shell
$ ./build-operator.sh
```

* 进入项目目录geaflow-kubernetes-operator下，通过Helm一键安装operator。

```shell
$ helm install geaflow-kubernetes-operator helm/geaflow-kubernetes-operator
```

![](https://pic1.zhimg.com/80/v2-f2eb3bb97a437df069d638b882cec1ab_1440w.png)

* 在K8S Dashboard中查看pod是否正常运行。

![](https://picx.zhimg.com/80/v2-034caab620e2b8c537585489fb881877_1440w.png)

### 提交作业

K8S Operator成功部署并运行后，就可以编写CR的YAML文件进行作业提交了。

```shell
$ kubectl apply geaflow-example.yml
```

这里使用项目内置示例作业举例，其YAML文件格式如下：

```yaml
apiVersion: geaflow.antgroup.com/v1
kind: GeaflowJob
metadata:
	# 作业名称
  name: geaflow-example
spec:
	# 作业使用的GeaFlow镜像
  image: geaflow:0.1
  # 作业拉取镜像的策略
  imagePullPolicy: IfNotPresent
  # 作业使用的k8s service account
  serviceAccount: geaflow
  # 作业java进程的主类
  entryClass: com.antgroup.geaflow.example.graph.statical.compute.sssp.SSSP
  clientSpec:
    # client pod相关的资源设置
    resource:
      cpuCores: 1
      memoryMb: 1000
      jvmOptions: -Xmx800m,-Xms800m,-Xmn300m
  masterSpec:
    # master pod相关的资源设置
    resource:
      cpuCores: 1
      memoryMb: 1000
      jvmOptions: -Xmx800m,-Xms800m,-Xmn300m
  driverSpec:
    # driver pod相关的资源设置
    resource:
      cpuCores: 1
      memoryMb: 1000
      jvmOptions: -Xmx800m,-Xms800m,-Xmn300m
    # driver个数
    driverNum: 1
  containerSpec:
    # container pod相关的资源设置
    resource:
      cpuCores: 1
      memoryMb: 1000
      jvmOptions: -Xmx800m,-Xms800m,-Xmn300m
    # container个数
    containerNum: 1
    # 每个container内部的worker个数(线程数)
    workerNumPerContainer: 4
  userSpec:
    # 作业指标相关配置
    metricConfig:
      geaflow.metric.reporters: slf4j
      geaflow.metric.stats.type: memory
    # 作业存储相关配置
    stateConfig:
      geaflow.file.persistent.type: LOCAL
		  geaflow.store.redis.host: host.minikube.internal
      geaflow.store.redis.port: 6379
    # 用户自定义参数配置
    additionalArgs:
      geaflow.system.state.backend.type: MEMORY
```

K8S环境上的作业强依赖于Redis组件，若你已经部署了Redis，则可以在geaflow-example.yaml中提供Redis主机和端口号。你也可以通过Docker快速启动一个本地Redis服务，默认地址host.minikube.internal可直接访问。

```shell
docker pull redis:latest
docker run -p 6379:6379 --name geaflow_redis redis:latest
```

#### 提交API任务

对于提交HLA任务的情况，需要额外注意以下几个参数：

- spec.entryClass：必填。
- spec.udfJars：选填，一般填写API任务的JAR文件的url地址。

```yaml
spec:
	# 必填
	entryClass: com.example.MyEntryClass
	# 可选
	udfJars: 
  	- name: myJob.jar
    	url: http://url-path-to-myJob.jar
```

#### 提交DSL任务

对于提交DSL任务的情况，需要额外注意以下几个参数：

- spec.entryClass：不填，留空（用于区分是API作业还是DSL作业）。
- spec.gqlFile：必填，请填写自己文件的名称和url地址。
- spec.udfJars：选填，如需UDF的话，请填写UDF JAR文件的url地址。

```yaml
spec:
	# 不填
	# entryClass: com.example.MyEntryClass
	# 必填
  gqlFile:
    # name必须填写正确，否则无法找到对应文件
    name: myGql.gql
    url: http://url-path-to-myGql.gql
	# 可选
	udfJars: 
  	- name: myUdf.jar
    	url: http://url-path-to-myUdf.jar
```

关于DSL任务和HLA任务的更多参数，我们在项目目录geaflow-kubernetes-operator/example目录中准备了两个demo作业供大家参考，请分别参考项目中的示例文件：

- example/example-dsl.yml
- example/example-hla.yml。


### 查看作业状态

可以访问K8S Dashboard查看pod是否被拉起，执行以下命令可以查看CR的状态是否已经正常运行。

```shell
$ kubectl get geaflowjob geaflow-example
```

![](https://pic1.zhimg.com/80/v2-7d7672e626ccc05609faa054f268a277_1440w.png)

![](https://pic1.zhimg.com/80/v2-e0b08c95899d015c341841a4cc40f82c_1440w.png)

若在提交过程中失败，则状态会变为FAILED。若需定位原因，可通过以下命令查看。

```shell
$ kubectl get geaflowjobs geaflow-example -o yaml
```

### 查看集群状态

Operator自带一个前端页面，可以展示集群的基本信息、所有作业的状态、错误信息、以及完整的配置，并做了分类统计。可以通过访问Operator的service或者pod的8089端口来打开页面。

![](https://pic1.zhimg.com/80/v2-1eb7dd2ed8f4959d1d7b744383168712_1440w.png)

**备注**：
> 在minikube环境中，需要通过portforward将Operator的pod代理到本地端口（默认为8089端口），请将operator-pod-name替换为实际的operator pod名称，然后通过浏览器访问localhost:8089即可打开页面。

```shell
$kubectl port-forward ${operator-pod-name} 8089:8089
```

**至此，我们完成了TuGraph Analytics作业的轻量级提交和运行！是不是超简单！快来试一试吧！**

------------------------

GeaFlow(品牌名TuGraph-Analytics) 已正式开源，欢迎大家关注！！！

欢迎给我们 Star 哦!

Welcome to give us a Star!

GitHub👉[https://github.com/TuGraph-family/tugraph-analytics](https://github.com/TuGraph-family/tugraph-analytics)

## 微信群
![image](../../../../assets/images/wechat.png)

请点击项目链接下方微信二维码添加微信用户群：[https://github.com/TuGraph-family/tugraph-analytics](https://github.com/TuGraph-family/tugraph-analytics)
