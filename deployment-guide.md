- [Taking Solr to Production](#taking-solr-to-production)
  - [Fine-Tune Your Production Setup](#fine-tune-your-production-setup)
    - [Override Settings in solrconfig.xml](#override-settings-in-solrconfigxml)
    - [Avoid Swapping (\*nix Operating Systems)](#avoid-swapping-nix-operating-systems)
      - [Disabling Swap](#disabling-swap)
      - [Reduce Swappiness](#reduce-swappiness)
  - [Security Considerations](#security-considerations)
- [SolrCloud Shards and Indexing](#solrcloud-shards-and-indexing)
  - [Leaders and Replicas](#leaders-and-replicas)
    - [Types of Replicas](#types-of-replicas)
    - [Combining Replica Types in a Cluster](#combining-replica-types-in-a-cluster)
- [SolrCloud Distributed Requests](#solrcloud-distributed-requests)
  - [Query Fault Tolerance](#query-fault-tolerance)
    - [zkConnected Parameter](#zkconnected-parameter)
    - [shard.tolerant Parameter](#shardtolerant-parameter)
    - [distrib.singlePass Parameter](#distribsinglepass-parameter)
  - [Distributed Inverse Document Frequency (IDF)](#distributed-inverse-document-frequency-idf)
- [ZooKeeper Ensemble Configuration](#zookeeper-ensemble-configuration)
  - [How Many ZooKeeper Nodes?](#how-many-zookeeper-nodes)
  - [Configuration for a ZooKeeper Ensemble](#configuration-for-a-zookeeper-ensemble)
    - [Initial Configuration](#initial-configuration)
    - [Ensemble Configuration](#ensemble-configuration)
    - [ZooKeeper Environment Configuration](#zookeeper-environment-configuration)
- [ZooKeeper Utilities](#zookeeper-utilities)
- [Configuring Logging](#configuring-logging)
  - [Logging Slow Queries](#logging-slow-queries)
  - [Selective Logging on SolrCore](#selective-logging-on-solrcore)
  - [Request Logging](#request-logging)
- [Configuring Authentication and Authorization](#configuring-authentication-and-authorization)
- [Configuring security.json](#configuring-securityjson)

# Taking Solr to Production

## Fine-Tune Your Production Setup

### Override Settings in solrconfig.xml
Solrの設定ファイルで `${sole.PROPERTY:DEFAULT_VALE}` という構文で書かれたプロパティはJavaのシステムプロパティで上書きできる。
例えば、デフォルトのソフトコミットの最大時間は以下のように設定されている。

```xml
<autoSoftCommit>
  <maxTime>${solr.autoSoftCommit.maxTime:3000}</maxTime>
</autoSoftCommit>
```

これを10秒に設定したい場合は、以下のようにSolrを起動する。

```bash
bin/solr start -Dsolr.autoSoftCommit.maxTime=10000
```

bin/solrスクリプトは起動時に-Dで始まるオプションをJVMに渡すだけで、本番環境ではインクルードファイル（例えば `/etc/default/solr.in.sh`）で定義された `SOLR_OPTS` 変数を使用するのがおすすめ

```bash
SOLR_OPTS="$SOLR_OPTS -Dsolr.autoSoftCommit.maxTime=10000"
```

### Avoid Swapping (*nix Operating Systems)
スワップが起きたとき、パフォーマンスが落ちた状態で処理を続けるより、ハードクラッシュさせて他のノードに引き継いだほうがよい。Linux環境ではスワップを無効化するか、"swappiness"を減らすのがおすすめ。

#### Disabling Swap
スワップを完全に無効化する場合はホストに十分な物理EAMがあることを確認する。

#### Reduce Swappiness
Linuxではデフォルトで"swappiness"が高い値に設定されているため、スワップに非常に積極的。これを非常に低い値に下げることでswapoff（一時的にスワップを無効化するコマンド）とほぼ同じ効果が得られる。

```bash
# 現在のswappinessの値を確認
cat /proc/sys/vm/swappiness

# swappinessを低い値に設定するには設定ファイルに以下の行を追加する
# vm.swappiness = 1
sudo vim /etc/sysctl.conf
```

## Security Considerations
Solrはデフォルトでループバックインターフェース(127.0.0.1)のみをリッスンする。これを変更するにはインクルードスクリプト（solr.in.shまたはsolr.in.cmd）に `SOLR_JETTY_HOST` を設定することで可能になる。また、システムプロパティ `-Dsolr.zk.embedded.host` でも設定可能。


https://solr.apache.org/guide/solr/latest/deployment-guide/taking-solr-to-production.html

# SolrCloud Shards and Indexing

## Leaders and Replicas

### Types of Replicas

| 名前               | トランザクションログの保存 | ローカルでのインデックス化 | リーダーになれる | 特徴                                                                                                         |
| ------------------ | -------------------------- | -------------------------- | ---------------- | ------------------------------------------------------------------------------------------------------------ |
| NRT (NearRealTime) | o                          | o                          | o                | デフォルトのタイプ                                                                                           |
| TLOG               | o                          | x                          | o                | レプリカ内でコミットしないのでインデックス作成を高速化できる。インデックスの更新はリーダーからの複製で行う。 |
| PULL               | x                          | x                          | x                | インデックスの更新はリーダーからの複製で行う。リーダー選挙に参加しない。                                     |

### Combining Replica Types in a Cluster
- すべてNRT: NRTタイプはソフトコミットをサポートする唯一のタイプなのでNearRealTimeが必要な場合に使う
- すべてTLOG: NearRealTimeが不要で、すべてのレプリカが更新リクエストを処理できるようにしたい場合に使う
- TLOG + PULL: NearRealTimeが不要で、一時的に古い検索結果を提供することになってもドキュメントの更新より検索クエリの可用性を高めたい場合に使う
- その他: 推奨されない

# SolrCloud Distributed Requests

## Query Fault Tolerance

### zkConnected Parameter
Solrノードは、リクエストを受信した時点でZooKeeperと通信できなくても、Solrノードが知っているすべてのシャードの少なくとも1つのレプリカと通信できる限り検索リクエストの結果を返す。これは通常、フォールトトレランスの観点から望ましい動作だが、ノードがZooKeeperから知らされていないコレクション構造の大きな変更（シャードの追加や削除、サブシャードへの分割など）があった場合、結果が古くなったり不正確になったりする可能性がある。

レスポンスヘッダに含まれる `zkConnected` は、リクエストを処理したノードがその時点でZooKeeperに接続しているかを表す値で、すべての検索レスポンスに含まれる。`sards.tolerant` パラメータに `requireZkConnected` を設定することで `zkConnected=false` のときにリクエストを失敗させることができる。 

### shard.tolerant Parameter
- `true` : 1つ以上の検索対象シャードが利用できない場合、部分的な結果を返し、レスポンスヘッダには `partialResults` というフラグが含まれる
- `false` : 1つ以上の検索対象シャードが利用できない場合、リクエストを失敗させる
- `requireZkConnected` : 1つ以上の検索対象シャードが利用できない場合か、検索結果を提供するノードがZooKeeperと通信できない場合にリクエストを失敗させる

### distrib.singlePass Parameter
trueにすると最初のフェーズですべてのフィールドをフェッチする。これは小さな値が入る非常に少数なフィールドを要求する場合に早くなる可能性がある。

## Distributed Inverse Document Frequency (IDF)
関連性の計算にはドキュメントとタームの統計情報が必要だが、分散検索だとこれはノードごとに異なる可能性があり、スコア計算が不正確になることがある。ドキュメントとタームの統計情報は `statsCache` に保存され、4つの実装がある。

- `LocalStatsCache` : ローカルの統計情報のみを関連性の計算に使用する。シャード間でタームの分布が一様な場合は合理的に機能する。`<statsCache>` が設定されていない場合はこれがデフォルト。
- `ExactStatsCache` : ドキュメントの頻度にコレクション全体のグローバルな値を使用する
- `ExactSharedStatsCache` : `ExactStatsCache` と似ているが、グローバルな統計情報は同じ条件のリクエストで再利用される（？）
- `LRUStatsCache` : 最も最近使われたキャッシュを利用する

設定は以下のように行う。
```xml
<statsCache class="org.apache.solr.search.stats.ExactStatsCache"/>
```

# ZooKeeper Ensemble Configuration

SolrにはApache ZooKeeperがバンドルされているが、ZooKeeperをホストするSolrノードがシャットダウンすると、ZooKeeperに依存しているシャードやSolrインスタンスがZooKeeperと通信できなくなるので、本番環境ではZooKeeper _ensemble_ をセットアップするべき。_ensemble_ とは、ZooKeeperを実行する複数のサーバのこと。

## How Many ZooKeeper Nodes?
- ensembleの原則はリクエストに応答するサーバの過半数を維持することで、この過半数を _quorum_ と呼ぶ
- quorumを適切に維持するためにensembleには奇数のZooKeeperを持つことが推奨される
- F台のZooKeeperの障害に耐えられるようにするには、2F+1台のZooKeeperが必要
- Solrクラスタが1000ノード規模でない限り、ZooKeeperは3ノードにとどめる
  - ノード間の調整が多くなり効率が悪くなるため

## Configuration for a ZooKeeper Ensemble

### Initial Configuration

`<ZOOKEEPER_HOME>/conf/zoo.cfg` に書く。このファイルはすべてのZooKeeperサーバに必要。

### Ensemble Configuration

同じく `<ZOOKEEPER_HOME>/conf/zoo.cfg` に書く。

この設定の中でサーバのホストとポートをIDと紐付けるが、各サーバがどのIDなのか示すため、サーバIDだけが書かれたファイル `/var/lib/zookeeper/<SERVER_ID>/myid` を作成する（IDが1ならファイルの中身は1のみ）

### ZooKeeper Environment Configuration

`<ZOOKEEPER_HOME>/conf/zookeeper-env.sh` に書く。このファイルはすべてのZooKeeperサーバに必要。

# ZooKeeper Utilities

- `zkcli.sh` : Solr固有のもので、ZooKeeperでSolrのデータを扱うためのもの
- `zkCli.sh` : ZooKeeperを操作するためのアプリケーションに依存しないシェル

# Configuring Logging

## Logging Slow Queries
スロークエリはINFOレベルでログに記録されるので、ログレベルをWARNにするとログに出力されないが、solrconfig.xmlのqueryセクションに `<slowQueryThresholdMillis>` を設定すると、しきい値より時間がかかるクエリはWARNレベルでログに出力される。スロークエリは `$SOLR_LOGS_DIR/solr_slow_requests.log` に出力される。

## Selective Logging on SolrCore
`log4j2.xml` の `MarkerFilter` で選択的にクエリのログ出力を無効化できる。

## Request Logging
すべてのHTTP(s)リクエストはデフォルトで標準NCSAフォーマットで `$SOLR_LOG_DIR/<yyyy_mm_dd>.request.log` 二出力され、毎日ロールオーバーされる。デフォルトでは3日分のリクエストログが保持される。環境変数または `solr.in.sh/solr.in.cmd` で `SOLR_REQUESTLOG_ENABLED=false` を設定することで、リクエストログを無効にできる。保持する日数はシステムプロパティ `-Dsolr.log.requestlog.retaindays` で変更できる。

# Configuring Authentication and Authorization

Solrには認証、認可、監査をサポートするフレームワークがあり、どのモードでも動作する。
関連する設定はすべて `security.json` という名前のファイルに保存する。
SolrCloudを使用する場合、このファイルはZooKeeper構造体のchrootに配置する必要がある。

# Configuring security.json

このファイルには、認証、認可、監査ロギングの各設定を行う3つのセクションがある。

```json
{
  "authentication" : {
    "class": "class.that.implements.authentication"
  },
  "authorization": {
    "class": "class.that.implements.authorization"
  },
  "auditlogging": {
    "class": "class.that.implements.auditlogging"
  }
}
```