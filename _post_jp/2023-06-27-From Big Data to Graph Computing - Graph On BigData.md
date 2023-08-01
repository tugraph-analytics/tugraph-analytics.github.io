---
layout: post
read_time: true
show_date: true
show_author: true
title: "ビッグデータからグラフ計算へ-Graph On BigData"
date: 2023-06-27
tags: [大数据, 图计算, TuGraph, GeaFlow, 开源, github]
category: opinion
author: 彭志伟
description: "ビッグデータからグラフ計算へ-Graph On BigData"
---

## 背景

2003年にGoogleの3つのビッグデータ分野のクラシック論文であるGFS、MapReduce、BigTableが発表されて以来、ビッグデータ領域は大きく発展してきました。特にオープンソースのビッグデータエンジンは、Hadoop、Hive、Storm、Spark、Flink、Prestoなどのさまざまな優れたオープンソースプロジェクトが次々と登場しました。これらのエンジンは、オフライン計算、ストリーム処理、OLAPクエリ、バッチストリームなど、さまざまな計算形態をカバーし、大規模データの処理技術がますます改善され多様化しています。

これらのビッグデータエンジンは主にテーブルモデルのデータを処理しており、処理するデータをテーブルモデルでモデリングして加工処理を行います。テーブルモデルは比較的単純で理解しやすいですが、複雑な関係演算や表現の処理にはかなりの困難さがあります。テーブルモデルは、テーブル間の関連を処理するためにJoinの方法を使用しますが、この方法は大量のシャッフルを引き起こし、実行リソースを増やします。特に関連度が高い場合、Joinの方法の欠点はより明確になります。また、最短パス、k-hopなどの複雑な関係の記述には、SQLのようなテーブルモデルの言語でも表現するのは難しいです。

グラフモデルは、点と辺を基本要素として定義するデータモデルとして、関連関係を自然に記述することができます。グラフモデルでは、点はエンティティを表し、辺は関係を表します。たとえば、ソーシャルネットワークの関係グラフでは、各人は1つの点で表され、人と人の関係は辺で表されます。人と人の間にはさまざまな複雑な関係が存在することがあり、これらの関係はさまざまな辺で表すことができます。グラフモデルをベースにすることで、複雑な関係とその演算をうまく記述できるだけでなく、グラフのストレージモデルは自然に点と辺の関係を格納し、計算面ではより優れた計算パフォーマンスを得ることができます。

![image](../../../../assets/images/posts/20230627/zhuanzhang.png)

## リアルタイムグラフ計算エンジン-GeaFlow

アリペイのリスク管理シナリオでは、環路の存在を確認するために、複数の転送関係を検索する必要があります。ログのアトリビューション分析のシナリオでは、ユーザーの行動パスを分析する必要があります。これらのシナリオでは、一方で関係が複雑であり、一方で計算のリアルタイム性が要求されます。ビジネスは通常、分単位または秒単位の遅延が必要です。また、グラフデータは大きく、数百億または数兆の点エッジスケールに達することがあります。従来のビッグデータエンジンは、Spark GraphXは大規模なグラフデータの処理能力を持っていますが、主にオフライン計算シナリオに特化しており、リアルタイム性の要件を満たすことはできません。Flinkは強力なリアルタイム計算能力を持っており、しかし、多跳のリアルタイムJoin関連計算、特に大規模なデータスケールのシナリオを扱うのが難しいです。

これらの問題と課題に直面したアリペイのグラフ計算チームは、実際の問題に基づいて、数年間の探索と実践の結果、分散リアルタイムグラフ計算エンジンGeaFlow（ブランド名TuGraph-Analytics）を実現しました。GeaFlowは、グラフモデルを基本的なデータモデルとし、グラフモデルを基にグラフ計算のプログラミングインターフェースを定義し、ストリーム処理能力と組み合わせて、ストリームグラフ計算の能力を実現しています。DSL言語のレベルでは、GeaFlowは表処理言語SQLとグラフクエリ言語ISO / GQLを組み合わせて、グラフと表を統合したデータ分析能力を実現しています。GeaFlowのストリームグラフ計算能力により、金融シーンでの大規模なデータの複雑な関連関係のリアルタイム計算の問題をうまく解決しています。

## GeaFlow全体のアーキテクチャ

GeaFlowの全体的なアーキテクチャは、次のレベルで構成されています：

![image](../../../../assets/images/posts/20230627/geaflow_arch.png)

* GeaFlow DSL: GeaFlowはユーザーにグラフとテーブルの統合分析言語を提供し、SQL + ISO / GQLの方式を採用しています。ユーザーは、SQLプログラミングのような方法でリアルタイムグラフ計算タスクを記述することができます。
* GraphView API: GeaFlowはGraphViewを中心に定義された一連のグラフ計算のプログラミングインターフェースであり、グラフの構築、グラフ計算、ストリームAPIインターフェースを含んでいます。
* GeaFlowランタイム: GeaFlowランタイムには、GeaFlowグラフオペレータ、タスクスケジューリング、フェイルオーバー、シャッフルなどのコア機能が含まれています。
* GeaFlowステート: GeaFlowのグラフの状態ストレージであり、グラフの頂点エッジデータを格納するために使用されます。また、ストリーム計算の状態、たとえば集約状態もステートに格納されます。
* K8Sデプロイメント: GeaFlowはK8Sを使用してデプロイおよび実行することができます。
* GeaFlowコンソール: GeaFlowの管理プラットフォームであり、ジョブ管理、メタデータ管理などの機能を含んでいます。

## GeaFlowとビッグデータエコシステムの統合
グラフ計算システムは孤立したシステムではなく、既存のビッグデータエコシステムと統合する必要があります。GeaFlowは、Connectorプラグインを介して主要なビッグデータエコシステムとの連携をサポートしています。例えば、Kafka/Hive/HDFSなどです。Connectorプラグインを使用することで、ビッグデータエコシステムのデータを簡単にグラフ計算システムに統合することができます。以下では、Hiveを例にとって、データウェアハウスのデータをGeaFlowのグラフストレージに取り込み、グラフアルゴリズムを実行する方法について説明します。

### グラフの定義

まず、グラフを定義する必要があります。以下のように、Create Graphステートメントを使用してグラフを定義します：
```sql
CREATE GRAPH IF NOT EXISTS friend (
  Vertex person (
    id bigint ID,
    name varchar
  ),
  Edge knows (
    srcId bigint SOURCE ID,
    targetId bigint DESTINATION ID,
    weight double
  )
) WITH (
  storeType='rocksdb',
  shardCount = 1
);
```
このグラフ定義には、点テーブルpersonとエッジテーブルknowsが含まれています。点テーブルpersonでは、点の属性情報とidフィールドが定義されており、idフィールドはグラフ内の点を一意に識別するためのもので、点テーブルの主キーです。idキーワードを使用して定義します。エッジテーブルknowsでは、友人関係が定義されており、srcIdは関係の始点を、SOURCE IDキーワードを使用して定義します。targetIdは関係の対象点を、DESTINATION IDキーワードを使用して定義します。weightフィールドはエッジの属性フィールドです。グラフの点テーブルまたはエッジテーブルは、0つ以上の属性フィールドを含むことができます。

### Hiveテーブルの定義

まず、Hiveの点テーブルとエッジテーブルを定義する必要があります。テーブルには、スキーマ情報やメタストアのURIなどが指定されています：

```sql
set geaflow.dsl.window.size = -1;

CREATE TABLE IF NOT EXISTS hive_person (
  id BIGINT,
  name VARCHAR
) WITH (
  type='hive',
  geaflow.dsl.hive.database.name = 'default',
  geaflow.dsl.hive.table.name = 'user',
  geaflow.dsl.hive.metastore.uris = 'thrift://localhost:9083'
);

CREATE TABLE IF NOT EXISTS hive_knows (
  src_id BIGINT,
  target_id BIGINT,
  weight DOUBLE
) WITH (
  type='hive',
  geaflow.dsl.hive.database.name = 'default',
  geaflow.dsl.hive.table.name = 'relation',
  geaflow.dsl.hive.metastore.uris = 'thrift://localhost:9083'
);
```
GeaFlowはストリームグラフ計算エンジンであり、データソースはwindow sizeで一連のwindowに分割され、エンジンはこれらのwindowのデータを順番に処理します。window sizeを-1に設定した場合、All Windowと呼ばれ、すべてのデータを一括処理します。Hiveなどのバッチデータソースインターフェースでは、window sizeを-1に設定して処理する必要があります。

### グラフの構築

グラフの構築は、外部データテーブルのデータをグラフに書き込むことによって行われます。Insertステートメントを使用してこれを行うことができます。以下のステートメントでは、hive_personテーブルとhive_knowsテーブルのデータをfriendグラフのpersonテーブルとknowsテーブルに書き込み、グラフデータの構築を完了します。

```sql
INSERT INTO friend.person(id, name)
SELECT
 id, name
FROM hive_person
;

INSERT INTO friend.knows
SELECT src_id, target_id, weight * 10
FROM hive_knows
;
```

### グラフ計算

次に、構築されたグラフデータに対してグラフアルゴリズムを実行します。ここでは、SSSP（単一始点最短経路）を例に説明します：

```sql
CREATE TABLE IF NOT EXISTS result (
  vid int,
	distance bigint
) WITH (
	type='file',
  `geaflow.file.persistent.config.json` = '{\'fs.defaultFS\':\'namenode:9000\'}',
	geaflow.dsl.file.path='/path/to/result'
);
-- 実行するグラフを定義
USE GRAPH friend;

INSERT INTO result
CALL SSSP(1) YIELD (vid, distance)
RETURN vid, distance
;
```

まず、計算結果を格納するresultテーブルを定義し、次にUSE GRAPHステートメントを使用して現在の計算で使用するグラフを設定します。最後に、CALLステートメントを使用してSSSPアルゴリズムを実行します（SSSPアルゴリズムの引数として始点のidを指定します）、そして計算結果をresultテーブルに書き込みます。

# まとめ

本文では、まずGeaFlowグラフ計算エンジンの背景について説明し、次にGeaFlowがどのようにビッグデータエコシステムと統合されるかについて説明しました。そして、Hiveのデータをグラフに変換してSSSPアルゴリズムを実行する方法を例に説明しました。

