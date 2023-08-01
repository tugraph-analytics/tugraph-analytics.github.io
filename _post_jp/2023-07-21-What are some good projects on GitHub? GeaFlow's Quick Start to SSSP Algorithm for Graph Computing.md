---
layout: post
read_time: true
show_date: true
show_author: true
title: "GitHubにはどのような優れたプロジェクトがありますか？GeaFlowグラフ計算の素早い入門：SSSPアルゴリズム"
date: 2023-07-21
tags: [蚂蚁集团, 图计算, 单源最短路径, GeaFlow, 开源, github]
category: news
author: TuGraph
description: "This article introduced the basic principles of the graph algorithm SSSP supported by the real-time graph computing engine GeaFlow, as well as its implementation details in GeaFlow, and demonstrated an application on the GitHub dataset."
---

Tips：この記事はChatGPT 3.5によって中国語から日本語に翻訳されました。不正確な点がある場合はお詫び申し上げます。

## 引言

以下の図は、GitHub内の約500個のオープンソースプロジェクトリポジトリとトピックが組み合わされた関係ネットワークです。密集した線は、誰も有用な情報を見つけることができない可能性があります。しかしながら、現在GitHubには3000000以上のリポジトリがあります！

![image](../../../../assets/images/posts/20230721/github1.png)

5分間で興味を持つ良いプロジェクトを見つける方法は？

今日は、GeaFlowの単一ソース最短経路アルゴリズム(SSSP)を利用して、盲人が象をさわっているようなものを試してみましょう！

GeaFlow(ブランド名TuGraph-Analytics)は、アリババグループがオープンソースで提供する分散型リアルタイムグラフ計算エンジンで、現在は金融リスク管理、ソーシャルネットワーク、ナレッジグラフ、データアプリケーションなどの場面で広く活用されています。

## SSSP(単一ソース最短経路アルゴリズム)の紹介

SSSP（単一ソース最短経路アルゴリズム）は、グラフ理論に基づいたアルゴリズムであり、出発点から他のすべてのノードへの最短経路を見つけるために使用されます。このアルゴリズムは、地図ナビゲーション、ネットワークトポロジーなど、さまざまな実際の問題に適用することができます。

GitHubのオープンソースプロジェクトリポジトリとトピックが組み合わされた関係ネットワークでは、リポジトリからトピックへ、そして再びリポジトリへと続くリンクは、SSSPアルゴリズムの実行をサポートすることができます。

![image](../../../../assets/images/posts/20230721/github2.png)

図に示すように、関係ネットワークのローカルでは、出発点からの距離をトピック/リポジトリまでの矢印の数でマークすることができます。たとえば、リポジトリFiraCodeとリポジトリFont-Awesomeの距離は2であり、2つの矢印で到達できます。これらは互いに最も近い関連リポジトリです。

簡単に言えば、私たちが興味を持っているリポジトリをマークし、私たちが興味を持っているリポジトリに最も近いリポジトリが推奨される良いリポジトリです。また、さらに進んで、STAR数が多い近距離のリポジトリは、より推奨されるかもしれません。

## GeaFlowによるSSSPの実現

SSSPアルゴリズムを実行するには、使用するグラフを指定し、クエリ内でグラフアルゴリズムを直接呼び出すことができます。構文形式は以下のようになります：

```sql
USE GRAPH github_repo_topic
INSERT INTO tbl_result
CALL sssp('source_vertex') YIELD (repoName, distance)
RETURN repoName, distance;
```

このコードは、グラフgithub_repo_topicで実行され、source_vertexをアルゴリズムの起点として、他のすべてのポイントの距離を出力します。結果が必要な場合、RETURNにWHERE条件フィルタを追加することができます。SQLクエリと同じです！

グラフアルゴリズムをカスタマイズする必要がある場合、AlgorithmUserFunctionインターフェイスを実装することができます。GeaFlowには、さまざまなグラフアルゴリズムの一般的な実装が組み込まれており、これらのアルゴリズムは個別にカスタマイズする必要はありません。例えば、SSSPアルゴリズムの参考実装は以下のようになります：

```java
@Description(name = "sssp", description = "built-in udga Single Source Shortest Path")
public class SSSP implements AlgorithmUserFunction<Object, Long> {

  private AlgorithmRuntimeContext<Object, Long> context;

  @Override
  public void init(AlgorithmRuntimeContext<Object, Long> context, Object[] parameters) {
    //Initialize the algorithm context.
    this.context = context;
  }

  @Override
  public void process(RowVertex vertex, Iterator<Long> messages) {
    long currentDistance;
    if (context.getCurrentIterationId() == 1L) {
      //Initialize the initial distance value for all vertices.
    } else if (context.getCurrentIterationId() <= $maxIteration) {
      //Calculate the shortest distance.
    } else {
      //Return the result.
    }
    //Update the distance value
    context.updateVertexValue(ObjectRow.create($currentDistance));
    //Send messages to neighbors
    context.sendMessage(vertex.getId(), $currentDistance);
    long scatterDistance = $currentDistance == Long.MAX_VALUE ? Long.MAX_VALUE : currentDistance + 1;
    for (RowEdge edge : context.loadEdges(EdgeDirection.OUT)) {
      context.sendMessage(edge.getTargetId(), scatterDistance);
    }
  }

  @Override
  public StructType getOutputType() {
    //Data type of the algorithm return value.
    return new StructType(
      new TableField("id", StringType.INSTANCE, false),
      new TableField("distance", LongType.INSTANCE, false)
    );
  }
}
```

グラフクエリは、課題を提出する形式で完了するため、課題はローカルまたはK8Sクラスタで実行することができ、GeaFlowはこれらのグラフ開発課題を管理し、トレースバックするためのコンソールを提供します。

![image](../../../../assets/images/posts/20230721/github3.png)

## GitHubの関係グラフ上でブラインドフォールディングを行う

話を長くせずに、現在GitHubで最もスター数が多いプロジェクトを見つけ、それと距離2（つまり共通のトピックを持つ）のプロジェクトを計算してみましょう。

現在、最もスター数が多いプロジェクトはfreeCodeCampです。データは、2023年5月までの[GitHub Public Repository Metadata](https://www.kaggle.com/datasets/pelmers/github-repository-metadata-with-5-stars)に基づいています。

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

短時間で計算結果を取得しましたので、GeaFlowがどのような良いプロジェクトを私たちに見つけたか見てみましょう。ここでは、スター数でソートしないで、10件のランダムなレコードを表示します。

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

## まとめ
本文では、リアルタイムグラフ計算エンジンGeaFlowがグラフアルゴリズムSSSPをサポートする基本原理とGeaFlow内での実装の詳細を紹介し、GitHubデータセット上でのアプリケーションを示します。




