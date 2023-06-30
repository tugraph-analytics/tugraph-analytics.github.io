---
layout: post
read_time: true
show_date: true
title: "谁在以太坊区块链上循环交易？TuGraph+Kafka的0元流图解决方案"
date: 2023-06-16
img: posts/20230630/kafka_cycle2.png
tags: [图计算, 以太坊, 循环交易, TuGraph-Analytics, 环路检测, github, Kafka]
category: opinion
author: 林力韬
description: "随着大数据和人工智能等技术的发展，即便人们都知道数据的价值日益凸显，却无法凭借一己之力获取和分析如此大规模的数据。"
---
author: 林力韬
co-author: ChatGPT 3.5

## 引言
都在说数据已经成为新时代的生产资料。

但随着大数据和人工智能等技术的发展，即便人们都知道数据的价值日益凸显，却无法凭借一己之力获取和分析如此大规模的数据。

要想富，先修路。要想利用新时代的数据致富，也必须要有趁手的工具。
只有合适的工具才能完成大规模数据的采集、清洗、存储、处理和可视化等各个环节。
只有具备这样的工具，才能更好地利用数据来推动经济社会发展，并为人类创造更多的价值。

![image](../../../../assets/images/posts/20230630/kafka_tugraph.png)

现如今，免费的，蚂蚁集团图计算团队开源的超大规模流图处理系统——TuGraph-Analytics。
正成为一款专注于图计算领域的开源数据处理工具。我们从一个简单的问题开始，让你体会TuGraph-Analytics的强大之处。

## “谁在进行循环交易？”

“谁在以太坊的区块链上进行循环交易？”

如何回答这个问题？

大数据领域的同学可能会立即举手，Kafka给我上，Flink给我上，实在不行Spark... 
这些系统在处理大规模数据时，工作流程通常非常长，开发效率较低，无法迅速地响应用户的需求。
各有各的长处，各有各的毛病。
因此，我们需要寻找全新的数据分析工具，以提高大数据领域的工作效率，并更好地应对日益复杂和多变的数据处理挑战。

新的工具？那么试试GPT？

Q：谁在以太坊的区块链上进行循环交易？

GPT：这个问题需要进行数据分析，涉及到大量的以太坊交易数据。
如果要确定谁在以太坊的区块链上进行循环交易，可以通过以下步骤：

获取以太坊的交易数据，包括交易哈希、发送方地址、接收方地址、交易类型和交易金额等信息。
对交易数据进行清洗和处理，去掉无效数据和重复数据等。
利用图计算等技术，构建以太坊交易网络，并对交易网络进行分析，识别出交易中的循环交易。
进一步分析循环交易的发送方地址和接收方地址，识别出谁在以太坊的区块链上进行循环交易。

需要注意的是，由于以太坊区块链的去中心化特性，交易数据的获取和分析可能存在一定的难度和挑战。

姿势不对，再来一次。

Q：谁在以太坊的区块链上进行循环交易？一句话回答

GPT：无法通过一句话回答，需要进行数据分析。

可见，虽然GPT等人工智能在自然语言处理和图像识别等方面已经取得了显著进展，但在数据洞察方面仍然存在挑战。
因为数据洞察往往需要结合具体的业务背景和领域知识，而这些人类经验和智慧难以完全被机器模拟和替代。此外，在数据清洗、数据预处理、特征提取和建模等关键步骤上，仍需要人类专家进行干预和优化。

今天，我们使用开源的TuGraph-Analytics与流式消息系统Kafka，
不仅能够很好回答谁在以太坊的区块链上进行循环交易，而且能够搭建秒级实时的金融级解决方案，客官且看----


## 何为循环交易？

来看这个例子

![image](../../../../assets/images/posts/20230630/kafka_001.png)

图中有7个不同颜色的点代表交易账户，每个账户对应以太坊上的一个钱包地址。
一个账户转出到另一个账户的过程，表示为图中有方向的边，这个图中共有8笔交易构成一个交易网络。

![image](../../../../assets/images/posts/20230630/kafka_002.png)

将交易头尾相连，能够找到返回起点的循环，就可以视为循环交易。

循环交易通常是指通过多个交易步骤，从而实现利用价格差异来获取利润的一种交易方式。
其过程包括在不同的市场或不同的时间段内进行买卖行为，从而获得利润。
在交易网络图中，可以使用环路检测算法来发现潜在的循环交易路径，并检索这条路径上的账户就能够知道哪些账户可能参与了循环交易。

可是，当一笔新的交易发生时——

![image](../../../../assets/images/posts/20230630/kafka_003.png)

可见，环路检测并不是一劳永逸的静态图算法。随着消息实时不断到来，交易网络图中的最新环路不断发生变化。
如果关心每一笔交易产生的交易环路，这将成为一个极其复杂的问题。

以太坊是基于区块链技术的智能合约平台，其交易速度取决于众多的参与者。
交易成交速度通常在15秒到1分钟之间，交易吞吐量可以达到每秒数十到数百笔交易。但别忘了，以太坊还有海量的历史交易数据，相当于存在一个相当复杂的”底图“，循环交易问题不仅需要考虑新的交易，还必须综合分析历史上所有的交易！

这就是为什么强如GPT也无法一句话回答这个问题的原因。

## TuGraph-Analytics

那么真的能够在每笔交易发生时实时地检测循环交易的产生吗？

实际上，使用专业的图计算系统TuGraph-Analytics，这个问题只需要几十行代码即可解决，还能够迅速结合Kafka搭建起交易听单->交易网络生成->实时循环交易检出->给下游发送消息完整的金融级实时解决方案。更关键的是，这一切都是免费的~ 感谢开源！

我们在TuGraph-Analytics新建一个图计算作业，只需大约40行代码，即可完成整个端到端流程。

![image](../../../../assets/images/posts/20230630/kafka_004.png)


代码描述如下，需要自取。

在这短短40行中，我们创建了以太坊交易的整个底图，命名为ethereum_transaction_network。
把来自Kafka的实时交易流table_new_trade不断添加到命名为ethereum_transaction_network的图中。

接下来，每当有新的交易到达的时刻，都将触发一次3跳循环交易模式的检查，
把更新的结果存入位于Kafka的外部表tbl_circular_trade，可以很方便地分发给下游组件。

实时交易数据我们采用了XBlock-ETH: Extracting and exploring blockchain data from Ethereum
这篇论文整理的真实的以太坊区块数据，在这个Demo中把以太坊地址隐去，以一个数字替代，即每个数字代表以太坊的一个钱包地址，相同的数字表示相同的地址。生成的循环路径结果我们拼接成一个字符串，方便观察。

```sql
set geaflow.dsl.window.size = 200;
-- 以太坊交易的底图
CREATE GRAPH IF NOT EXISTS ethereum_transaction_network (
	Vertex address (id bigint ID),
	Edge trade (srcId bigint SOURCE ID, targetId bigint DESTINATION ID, mount double)
) WITH (
	storeType='rocksdb'
);

-- 从Kafka监听最新的交易
CREATE TABLE IF NOT EXISTS table_new_trade (
  srcId bigint, targetId bigint, mount double
) WITH (
	type='kafka',
  geaflow.dsl.kafka.servers = '{your.kafka.server.ip}:9092',
	geaflow.dsl.kafka.topic = 'tbl-trade'
);

-- 将新的交易添加到底图
INSERT INTO ethereum_transaction_network.address SELECT srcId FROM table_new_trade;
INSERT INTO ethereum_transaction_network.address SELECT targetId FROM table_new_trade;
INSERT INTO ethereum_transaction_network.trade SELECT srcId, targetId, mount FROM table_new_trade;

-- 一张外部表连接Kafka以输出检出的循环交易结果
CREATE TABLE IF NOT EXISTS tbl_circular_trade (
	  circular VARCHAR
) WITH (
	type='kafka',
  geaflow.dsl.kafka.servers = '{your.kafka.server.ip}:9092',
	geaflow.dsl.kafka.topic = 'tbl-circular-trade'
);

-- 使用以太坊交易图进行实时查询
USE GRAPH ethereum_transaction_network;

-- 查询3跳循环交易模式存入位于Kafka的外部表
INSERT INTO tbl_circular_trade
MATCH (v1)-[:trade]->(v2)-[:trade]->(v3)-[:trade]->(v4)
WHERE v1.id = v4.id AND v1.id != v2.id AND v1.id != v3.id AND v2.id != v3.id
RETURN concat(CAST(v1.id as VARCHAR), '->', CAST(v2.id as VARCHAR),
       '->', CAST(v3.id as VARCHAR), '->', CAST(v4.id as VARCHAR)) AS circular
;
```

## 实机演示！

打开一个Kafka Producer，产生消息流，将交易不断发送给Kafka，如左侧终端窗口所示。
平均1秒左右，最新的图中循环交易检出结果就打印在右侧的Kafka Consumer窗口中。

![image](../../../../assets/images/posts/20230630/kafka_005.png)


当添加一些新的交易日志时，右侧的Kafka Consumer窗口中也实时更新了新的循环交易检出结果，响应十分迅速。

![image](../../../../assets/images/posts/20230630/kafka_006.png)


以上便是开源图计算平台TuGraph Analytics结合Kafka，快速搭建的以太坊循环交易检出解决方案。
虽然这只是一个小小的demo，但实机演示中强大的图计算语法和系统性能可见一斑，大大较低了我们普通人搭建图计算应用的难度和成本。

说多不如现在开始行动，TuGraph Analytics是你不容错过的利器。现在就赶快访问[https://github.com/TuGraph-family/tugraph-analytics](https://github.com/TuGraph-family/tugraph-analytics)，亲自体验TuGraph Analytics的魅力吧！此外，我们欢迎各位参与到开源贡献中来，为社区贡献自己的智慧和代码，共同促进国产图计算系统的发展。

## 参考文献
P. Zheng, Z. Zheng, J. Wu, and H.-N. Dai, “XBlock-ETH: Extracting and exploring blockchain data from Ethereum,” IEEE Open J. Comput. Soc., vol. 1, pp. 95–106, May 2020, doi: 10.1109/OJCS.2020.2990458.


GeaFlow(品牌名TuGraph-Analytics) 已正式开源，欢迎大家关注！！！

欢迎给我们 Star 哦!

Welcome to give us a Star!

GitHub:[https://github.com/TuGraph-family/tugraph-analytics](https://github.com/TuGraph-family/tugraph-analytics)

### 微信群
![image](../../../../assets/images/wechat.png)

请点击项目链接下方微信二维码添加微信用户群：[https://github.com/TuGraph-family/tugraph-analytics](https://github.com/TuGraph-family/tugraph-analytics)

