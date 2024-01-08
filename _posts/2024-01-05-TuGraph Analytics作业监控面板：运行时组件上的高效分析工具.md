---
layout: post
read_time: true
show_date: true
show_author: true
title: "TuGraph Analytics作业监控面板：运行时组件上的高效分析工具"
date: 2023-12-13
tags: [图计算, 开源, K8S, 监控面板, 蚂蚁集团, 分析]
category: news
author: 丁一
description: "TuGraph Analytics作业监控面板：运行时组件上的高效分析工具"
---

作者：丁一

## 背景

TuGraph Analytics作业部署到K8S集群之后，通常会启动多个pod（一个master、一/多个driver、多个container）。用户很难判断作业当前运行的进度如何，也不能通过pod的状态来判断内部进程的状态。无论是查看进度、查看日志、性能分析，都需要到每一个pod中进行对应的操作，运维成本很大，需要一个白屏化的监控页面来监控所有进程的实时状态信息。

因此，我们在作业进程中内置了一个Dashboard（本地启动/容器启动时自动生效），包括前端页面和后端server，用户可以不需要感知到它们的存在。通过访问Dashboard，用户可以更方便地通过白屏化的方式查看作业的执行进度、组件列表和详情、任意组件内部的指标、日志等。还可以通过Profiler工具对进程状态进行分析，快速定位问题。

## Dashboard介绍

TuGraph Analytics的Dashboard模块提供了作业级别的监控页面，可以轻松地查看作业的以下信息：

- 作业的健康度（Container和Worker活跃度）
- 作业的进度（Pipeline和Cycle信息）
- 作业各个组件的实时日志
- 作业各个组件的进程指标
- 作业各个组件的火焰图
- 作业各个组件的Thread Dump

## 如何访问页面

页面的服务部署在master组件上，因此直接访问master组件的地址即可（默认端口8090）。

![访问Dashboard](https://picx.zhimg.com/80/v2-048863dc1d5369f3812db1e57dc27929_1440w.png)

## 功能介绍

TuGraph Analytics Dashboard包含以下几个主要的功能：

### Overview

Overview页面会展示整个作业的健康状态。你可以在这里查看container和driver是否都在正常运行。

![概览](https://picx.zhimg.com/80/v2-1a91cb5ee21b1ec15a6db1a21e8b34a4_1440w.png)

除此之外，Overview页面也会展示作业的Pipeline列表。

### 作业执行计划进度

作业的执行计划可以由多个Pipeline表示，每个Pipeline内部又有多个Cycle。
可以通过侧边栏的Pipeline菜单进入页面。页面包括作业的每一项Pipeline的名称、开始时间和耗时。
耗时为0表示该Pipeline已开始执行，但尚未完成。

![执行计划-Pipeline](https://pic1.zhimg.com/80/v2-10d433b66e6f567a8b3094a1c2ea8feb_1440w.png)

点击Pipeline名称可以进入二级菜单，查看当前Pipeline下所有的Cycle列表的各项信息。

![执行计划-Cycle](https://pic1.zhimg.com/80/v2-c86c55ad7fa97af34017f0f42eb96aec_1440w.png)

### 作业组件详情

可以查看作业的各个组件（包括master、driver、container）的各项信息。可以通过侧边栏的菜单进行访问。
其中Driver详情展示所有driver的基础信息。

![Driver信息](https://picx.zhimg.com/80/v2-23db8291c5405118b8737cfb410d8188_1440w.png)

Container详情展示所有Container的基础信息。

![Container信息](https://picx.zhimg.com/80/v2-4546a6f511207e9e2e5c100558c3b7a4_1440w.png)

## 组件运行时详情

通过点击左边栏的Master详情，或者通过点击Driver/Container详情中的组件名称，可以跳转到组件的运行时页面。在运行时页面中，可以查看和操作以下内容。

### 进程指标

展示完整的容器进程指标。

![进程](https://pic1.zhimg.com/80/v2-7be5e0ac15e5d5fe1927ed5b167fef6b_1440w.png)

### 容器日志

展示容器进程内的主要可见日志。
根据日志的log4j配置，默认日志文件大小最大为128G（此处测试简单起见设置为了50KB），超过后会进行文件备份。例如master.log.1和master.log.2就是master.log的备份之一。

![日志](https://pica.zhimg.com/80/v2-e114bfe71e6cafc7484a4c078aac4797_1440w.png)

- master.log：Master的java主进程日志。
- master.log.1 / master.log.2：Master的java主进程日志备份。
- agent.log：Master的agent服务日志。
- geaflow.log：进入容器后的shell启动脚本日志。

点击任意一个日志可以进入日志详情页面。日志的获取进行了后端分页，可以在右下角选择每页的KB大小，并可以跳转到指定页数。

![日志详情](https://picx.zhimg.com/80/v2-fb9dbabe6bc92de221a42cbc8fc5b794_1440w.png)

### 火焰图

展示火焰图的历史执行结果，并可重新生成新的火焰图。火焰图分析类型可选择CPU或ALLOC，单次最多分析60秒，最多保留10份历史记录。
点击“新建”，即可生成新的火焰图。

- 火焰图类型：可选CPU或者ALLOC（Memory）。
- 执行时间：分析时间，需介于1~60秒之间。

![新建火焰图](https://picx.zhimg.com/80/v2-8cf346a007990c7aff9c1ca940739d1b_1440w.png)

火焰图的执行时间根据用户的选择可能较久，因此会在后台静默执行。需要等待执行结束后，手动点击“新建”按钮旁边的“刷新”标识，获取最新的火焰图历史。

![火焰图](https://picx.zhimg.com/80/v2-1e9894f54d05e1aabd0cba3492c05425_1440w.png)

![火焰图详情](https://picx.zhimg.com/80/v2-7814b9dde2a5e7e0473ee5f909cc3418_1440w.png)

### Thread Dump

展示主进程的Thread Dump结果，并可重新进行Dump。保留最新一次dump的结果。

![Thread Dump](https://pic1.zhimg.com/80/v2-08e57b14209c56f41c59d1a162cd8a79_1440w.png)

点击“重新执行”，等待执行结束后，结果会自动刷新。
![Thread Dump详情](https://picx.zhimg.com/80/v2-9431d48c43bde3ed573de23d48d844c1_1440w.png)

### 进程配置

展示master的java主进程内的各项配置（仅master拥有此页面）。

![进程配置](https://picx.zhimg.com/80/v2-0ae957c82dd518f36d68850acbade5dd_1440w.png)

## 其他用法

### 列表排序与查询

部分列表的列可以进行排序和查询。
查询时，点击“搜索”标识，输入关键字，点击“搜索”按钮即可。
重置时，点击“重置”按钮，列表会重新刷新。

![列表](https://picx.zhimg.com/80/v2-b500aafe8c7c81e14daa5306eca80407_1440w.png)

### 国际化

页面支持中英文切换，点击右上角的“文A”图标，即可选择语言。

![国际化](https://picx.zhimg.com/80/v2-2e9df77f71690cc4e921f17818a56925_1440w.png)


**欢迎关注我们的GitHub仓库：**  [**https://github.com/TuGraph-family/tugraph-analytics**](https://github.com/TuGraph-family/tugraph-analytics)

------------------------

GeaFlow(品牌名TuGraph-Analytics) 已正式开源，欢迎大家关注！！！

欢迎给我们 Star 哦!

Welcome to give us a Star!

GitHub👉[https://github.com/TuGraph-family/tugraph-analytics](https://github.com/TuGraph-family/tugraph-analytics)

## 微信群
![image](../../../../assets/images/wechat.png)

请点击项目链接下方微信二维码添加微信用户群：[https://github.com/TuGraph-family/tugraph-analytics](https://github.com/TuGraph-family/tugraph-analytics)
