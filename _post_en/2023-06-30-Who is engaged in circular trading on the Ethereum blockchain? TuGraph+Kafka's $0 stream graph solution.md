---
layout: post
read_time: true
show_date: true
title: "Who is engaged in circular trading on the Ethereum blockchain? TuGraph+Kafka's $0 stream graph solution"
date: 2023-06-16
tags: [图计算, 以太坊, 循环交易, GeaFlow, 环路检测, github, Kafka]
category: opinion
author: 林力韬
description: "Who is engaged in circular trading on the Ethereum blockchain? TuGraph+Kafka's $0 stream graph solution"
---
author: 林力韬
co-author: ChatGPT 3.5

## Introduction
Everyone is talking about how data has become the new means of production in the new era. 

However, even though people are aware of the increasing value of data with the development of technologies like big data and artificial intelligence, they cannot obtain and analyze such massive amounts of data on their own.

To become rich, one must build roads. Similarly, to utilize the wealth of data in the new era, one must have suitable tools. Only with the right tools can one complete the various stages of data collection, cleansing, storage, processing, and visualization on a large scale. Only with these tools can one better utilize data to drive economic and social development, and create more value for humanity.

![image](../../../../assets/images/posts/20230630/kafka_tugraph.png)

Nowadays, TuGraph, a super large-scale stream graph processing system, developed by the Alibaba Group's Graph Computing Team, has become an open-source data processing tool dedicated to graph computation. We will start with a simple question to demonstrate the power of TuGraph.

## "Who is engaged in circular trading?"

"Who is engaged in circular trading on the Ethereum blockchain?"

How do we answer this question?

Those familiar with the field of big data may immediately raise their hands and say, "Bring me Kafka, Flink, and if necessary, Spark..." However, these systems often have long workflows and low development efficiency when dealing with large-scale data, making it difficult to respond quickly to user demands.

Each system has its strengths and weaknesses. Therefore, we need to find new data analysis tools to improve the efficiency of work in the field of big data and better address the increasingly complex and ever-changing challenges of data processing.

New tools? How about trying GPT?

Q: Who is engaged in circular trading on the Ethereum blockchain?

GPT: This question requires data analysis and involves a large amount of Ethereum transaction data. To determine who is engaged in circular trading on the Ethereum blockchain, you can follow these steps:
Retrieve Ethereum transaction data, including transaction hash, sender address, recipient address, transaction type, and transaction amount. Clean and process the transaction data, removing invalid and duplicate data. Use graph computing and other technologies to construct the Ethereum transaction network and analyze it to identify circular trading within the transactions. Further analyze the sender and recipient addresses involved in circular trading to identify who is engaged in circular trading on the Ethereum blockchain.

It is worth noting that due to the decentralized nature of the Ethereum blockchain, acquiring and analyzing transaction data may present certain difficulties and challenges.

Wrong approach, let's try again.

Q: Who is engaged in circular trading on the Ethereum blockchain? One sentence answer.
GPT: It cannot be answered in one sentence; data analysis is required.

As we can see, although artificial intelligence, such as GPT, has made significant progress in natural language processing and image recognition, challenges still exist in data insight. This is because data insight often requires the combination of specific business context and domain knowledge, which human experience and wisdom are difficult to fully simulate and replace. Additionally, human experts are still needed for intervention and optimization in critical steps such as data cleansing, preprocessing, feature extraction, and modeling.

Today, by using the open-source TuGraph and the stream messaging system Kafka, we can not only answer who is engaged in circular trading on the Ethereum blockchain but also build real-time financial-grade solutions that operate in milliseconds. Ladies and gentlemen, please take a look.

## What is circular trading?

Let's look at this example.

![image](../../../../assets/images/posts/20230630/kafka_001.png)

In the graph, there are 7 points of different colors representing trading accounts, with each account corresponding to a wallet address on the Ethereum platform.

The process of one account transferring to another is represented by directed edges in the graph, and there are a total of 8 transactions forming a transaction network in this graph.

![image](../../../../assets/images/posts/20230630/kafka_002.png)

Connecting the head and tail of transactions can identify cycles that return to the starting point, which can be considered as circular trading. 

Circular trading typically refers to a trading strategy that exploits price differentials for profit through multiple transaction steps. This process involves buying and selling in different markets or at different times to gain profits. In a transaction network graph, a cycle detection algorithm can be used to discover potential circular trading paths and retrieve the accounts involved in these paths to identify participants in circular trading.

However, when a new transaction occurs--

![image](../../../../assets/images/posts/20230630/kafka_003.png)

As we can see, cycle detection is not a one-time static graph algorithm. With real-time incoming messages, the latest cycles in the transaction network graph are constantly changing. If we are concerned about the cycles generated by each transaction, it becomes an extremely complex problem.

Ethereum is a smart contract platform based on blockchain technology, and its transaction speed depends on numerous participants. Transaction settlement speed typically ranges from 15 seconds to 1 minute, with transaction throughput reaching tens to hundreds of transactions per second. However, let's not forget that Ethereum also has a massive amount of historical transaction data, which can be considered as a complex "underlying graph." Therefore, the circular trading problem not only requires considering new transactions but also comprehensive analysis of all historical transactions.

This is why even powerful models like GPT cannot provide a one-sentence answer to this question.

## TuGraph-Analytics

Is it really possible to detect the occurrence of circular trading in real-time with each transaction?

In reality, using the professional graph computing system TuGraph-Analytics, this problem can be solved with just a few dozen lines of code. It can also be quickly integrated with Kafka to build a complete real-time financial-grade solution, including transaction listening, transaction network generation, real-time circular trading detection, and sending messages to downstream systems. The best part is that all of this is free, thanks to open-source!

In TuGraph, we can create a graph computing job with just around 40 lines of code to complete the entire end-to-end process.

![image](../../../../assets/images/posts/20230630/kafka_004.png)

The code is described as follows, feel free to take it.

In these short 40 lines, we create the entire underlying graph of Ethereum transactions, named "ethereum_transaction_network." We continuously add real-time transaction flows from Kafka, named "table_new_trade," to the graph named "ethereum_transaction_network."

Next, whenever a new transaction arrives, it triggers a check for a 3-hop circular trading pattern. The updated results are stored in an external table called "tbl_circular_trade," located in Kafka, which can be conveniently distributed to downstream components.

For real-time transaction data, we used actual Ethereum blockchain data extracted and explored from the paper "XBlock-ETH: Extracting and exploring blockchain data from Ethereum." In this demo, we have hidden the Ethereum addresses and replaced them with numbers, where each number represents a wallet address on Ethereum, and the same number represents the same address. The generated circular path results are concatenated into a string for easy observation.

```sql
set geaflow.dsl.window.size = 200;
-- Underlying graph of Ethereum transactions
CREATE GRAPH IF NOT EXISTS ethereum_transaction_network (
	Vertex address (id bigint ID),
	Edge trade (srcId bigint SOURCE ID, targetId bigint DESTINATION ID, mount double)
) WITH (
	storeType='rocksdb'
);

-- Listen to the latest transactions from Kafka
CREATE TABLE IF NOT EXISTS table_new_trade (
  srcId bigint, targetId bigint, mount double
) WITH (
	type='kafka',
  geaflow.dsl.kafka.servers = '{your.kafka.server.ip}:9092',
	geaflow.dsl.kafka.topic = 'tbl-trade'
);

-- Add new transactions to the underlying graph
INSERT INTO ethereum_transaction_network.address SELECT srcId FROM table_new_trade;
INSERT INTO ethereum_transaction_network.address SELECT targetId FROM table_new_trade;
INSERT INTO ethereum_transaction_network.trade SELECT srcId, targetId, mount FROM table_new_trade;

-- An external table connects to Kafka to output detected circular trading results
CREATE TABLE IF NOT EXISTS tbl_circular_trade (
	  circular VARCHAR
) WITH (
	type='kafka',
  geaflow.dsl.kafka.servers = '{your.kafka.server.ip}:9092',
	geaflow.dsl.kafka.topic = 'tbl-circular-trade'
);

-- Perform real-time queries using the Ethereum transaction graph.
USE GRAPH ethereum_transaction_network;

-- Perform a query for the 3-hop circular trading pattern and store the results in an external table located in Kafka.
INSERT INTO tbl_circular_trade
MATCH (v1)-[:trade]->(v2)-[:trade]->(v3)-[:trade]->(v4)
WHERE v1.id = v4.id AND v1.id != v2.id AND v1.id != v3.id AND v2.id != v3.id
RETURN concat(CAST(v1.id as VARCHAR), '->', CAST(v2.id as VARCHAR),
       '->', CAST(v3.id as VARCHAR), '->', CAST(v4.id as VARCHAR)) AS circular
;
```

## Live demonstration!

Open a Kafka Producer and generate a stream of messages by continuously sending transactions to Kafka, as shown in the left terminal window. On average, within approximately 1 second, the latest detected circular trading results in the graph will be printed in the Kafka Consumer window on the right.

![image](../../../../assets/images/posts/20230630/kafka_005.png)


As new transaction logs are added, the Kafka Consumer window on the right also updates in real-time with the newly detected circular trading results. The response is very prompt.

![image](../../../../assets/images/posts/20230630/kafka_006.png)


The above is the Ethereum circular trading detection solution quickly built using the open-source graph computing platform TuGraph integrated with Kafka. Although this is just a small demo, the powerful graph computing syntax and system performance demonstrated in the live demonstration provide a glimpse into significantly reducing the difficulty and cost of building graph computing applications for ordinary people.

Actions speak louder than words, so let's get started now. TuGraph is a tool you can't afford to miss. Visit [https://github.com/TuGraph-family/tugraph-analytics](https://github.com/TuGraph-family/tugraph-analytics) to experience the charm of TuGraph firsthand! Additionally, we welcome everyone to contribute to the open-source community, sharing their wisdom and code to collectively promote the development of domestic graph computing systems.

## References
P. Zheng, Z. Zheng, J. Wu, and H.-N. Dai, “XBlock-ETH: Extracting and exploring blockchain data from Ethereum,” IEEE Open J. Comput. Soc., vol. 1, pp. 95–106, May 2020, doi: 10.1109/OJCS.2020.2990458.
