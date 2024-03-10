# Table of Contents

- [Table of Contents](#table-of-contents)
- [Solr Configuration Files](#solr-configuration-files)
- [Index Location and Format](#index-location-and-format)
- [Index Segments and Merging](#index-segments-and-merging)
  - [Merging Index Segments](#merging-index-segments)
    - [Controlling Segment Sizes](#controlling-segment-sizes)
- [Commits and Transaction Logs](#commits-and-transaction-logs)
  - [\<updateHandler\> in solrconfig.xml](#updatehandler-in-solrconfigxml)
  - [commits](#commits)
    - [Hard Commits vs. Soft Commits](#hard-commits-vs-soft-commits)
    - [Explicit Commits](#explicit-commits)
    - [Automotic Commits](#automotic-commits)
  - [Transaction Log](#transaction-log)
- [Caches and Query Warming](#caches-and-query-warming)
  - [Caches](#caches)
    - [Filter Cache](#filter-cache)
    - [Query Result Cache](#query-result-cache)
    - [Document Cache](#document-cache)
    - [User Defined Caches](#user-defined-caches)
- [Request Handlers and Search Components](#request-handlers-and-search-components)
  - [Defining and Calling Request Handlers](#defining-and-calling-request-handlers)
  - [Defaults, Appends, and Invariants](#defaults-appends-and-invariants)
    - [Defaults](#defaults)
    - [Appends](#appends)
    - [Invariants](#invariants)
  - [InitParams](#initparams)
  - [Paramsets and UseParams](#paramsets-and-useparams)
- [Update Request Processors](#update-request-processors)
  - [URP Anatomy and Lifecycle](#urp-anatomy-and-lifecycle)

# Solr Configuration Files

Solrにはいくつかの設定ファイルがある。
- `solr.xml`
  - Solrサーバインスタンスの設定ファイル
  - JVM起動時に `-D` で指定されるJVMのシステムプロパティを使用できる
  - `$SOLR_HOME` ディレクトリにある（SolrCloudを使っている場合はZooKeeperに置くこともできるが非推奨）
  - デフォルトの設定は `$SOLR_TIP/server/solr/solr.xml` にある

以下はコアごとに存在する。
- `core.properties` : 各コアに固有のプロパティを定義する（コアの名前、コアが所属するコレクション、スキーマの場所など）
- `solrconfig.xml` : 高レベルの動作を制御する（データディレクトリの場所など）
- `managed-schema.xml` or `schema.xml` : ドキュメントのフィールドとフィールドタイプを定義する

# Index Location and Format

インデックスのデータはデフォルトではコアのインスタンスディレクトリの下の `/data` に置かれるが、`core.properties` の `dataDir` か `solrconfig.xml` の `<dataDir>` パラメータで変更できる（どっちが優先？）
また、環境変数 `SOLR_DATA_HOME` が定義されているか、`DirectoryFactory` に `solr.data.home` が設定されているか、`solr.xml` に `<solrDataHome>` 要素が含まれる場合、データディレクトリは `<SOLR_DATA_HOME>/インスタンス名/data` になる（結局どれが優先？）

# Index Segments and Merging

## Merging Index Segments

### Controlling Segment Sizes

ユーザは一度にマージするセグメントの数やマージするセグメントの最大サイズを変更できるが、これらの設定は検索とインデックス作成の速度のトレードオフとなる。
インデックスに含まれるセグメントが少なければ検索は高速になるが、マージが頻繁に起こるのでインデックスの作成は遅くなる。

# Commits and Transaction Logs

## \<updateHandler> in solrconfig.xml

コミット戦略の設定は `<updateHandler>` 要素で構成される。この要素はclassパラメータを取り、値は `solr.DirectUpdateHandler2` でなければならない。

## commits

Solrに送られたデータはインデックスにコミットされるまで検索できない。

### Hard Commits vs. Soft Commits

Solrはハードコミットとソフトコミットの2種類のコミットをサポートしている。

ハードコミット
- インデックスファイルに対してfsyncを呼び出し、ストレージにフラッシュされたことを確認する
- 現在のトランザクションログが閉じられ、新しいログが開かれる
- ハードコミットが行われなかったデータの復旧方法は後述のTransaction Logを参照
- 任意のタイミングでハードコミットすることでドキュメントを検索可能にできるが、ソフトコミットよりコストが高い
- デフォルトのコミットはハードコミット

ソフトコミット
- インデックスの変更を可視化（？）するだけで、インデックスファイルのfsync、新しいセグメントの開始、新しいトランザクションログの開始を行わないので高速

サーバがクラッシュした場合、ハードコミットはSolrがデータの保存場所を正確に知っていることを意味する。ソフトコミットはデータは保存されているが、位置情報はまだ保存されていないことを意味する。トレードオフとして、ソフトコミットはバックグラウンドでのマージが終わるの待たずに済むので、より高速な可視性が得られる。

これってソフトコミットは検索結果には反映されるようにはなるけど、ディスクには保存されないってこと？

### Explicit Commits

- `commit=true` : 更新リクエストにこのパラメータが含まれると、インデックスの更新完了と同時に、更新の追加と削除の影響を受けるすべてのインデックスセグメントがディスクに書き込まれる
- `softCommit=true` : 更新リクエストにこのパラメータが含まれると、ソフトコミットを実行する

### Automotic Commits

`solrconfig.xml` で `autoCommit` を設定することでコミットのタイミングを制御できる。しきい値のうち、最初に達したものがコミットのトリガーになる。`autoSoftCommit` を使えばソフトコミットのタイミングも制御できる。

## Transaction Log

最後のハードコミット以降の更新を記録する。ハードコミットが発生するたびに現在のログが閉じられ、新しいログが開かれる。ソフトコミットはトランザクションログに影響を与えない。

グレースフルでないシャットダウン（電源喪失、JVMのクラッシュ、kill -9など）の場合、ハードコミットされていなくてもtlogに記録されていればデータは失われず、起動時に再生される。グレースフルシャットダウンの場合、Solrはtlogとインデックスセグメントを閉じるので起動時に再生する必要はない。

# Caches and Query Warming

## Caches

SolrのキャッシュはIndex Searcherの特定のインスタンスに関連付けられていて、そのsercherの有効期間中は変更されないインデックスの特定のビュー。
デフォルトでは、キャッシュされたSolrオブジェクトは時間が経過しても期限切れにならない。

新しいsercherが開かれると、現在のsercherのキャッシュを利用してキャッシュを自動更新して自分自身に登録する。この間は現在のsercherがリクエストを受ける。新しいサーチャーの準備が整うと、現在のサーチャーとして登録されリクエストの処理を開始する。古いサーチャーは、すべてのリクエストを処理し終えるとクローズされる。

### Filter Cache

パースされたクエリとマッチするすべてのドキュメントの非順序集合を保持する。
集合が些細なものでなければ、実装はbitset。
最も典型的な使用法は各fq検索パラメータの結果をキャッシュすること。

### Query Result Cache

以前の検索結果（リクエストされたクエリ、ソート、ドキュメントの範囲に基づいたドキュメントIDの順序付きリスト（DocList））を保持する。

### Document Cache

Luceneのドキュメントオブジェクトを保持する。

### User Defined Caches

独自の名前付きキャッシュを定義することもできる。
キャッシュの名前を使って `IndexSearcher` の `getCache()`, `cacheLookup()`, `cacheInsert()` を呼んで使う。
auto-warmingを使用したい場合は、`solr.search.CacheRegenerator` を実装するクラスを `regenerator` パラメータで指定する。

# Request Handlers and Search Components

## Defining and Calling Request Handlers

- `/select` : クエリを処理するデフォルトのリクエストハンドラ
- `/update` : インデックス更新を処理するデフォルトのリクエストハンドラ

リクエストハンドラを設定する方法は3つある。

## Defaults, Appends, and Invariants

1つ目の方法は `<sequestHnadler>` で定義する。

### Defaults

最も一般的な方法は `<sequestHnadler>` 要素の子要素として `<lst name="defaults">` を定義すること。
ここで定義されたパラメータは上書きされない限り常に使われる。

```xml
<requestHandler name="/select" class="solr.SearchHandler">
  <lst name="defaults">
    <str name="echoParams">explicit</str>
    <int name="rows">10</int>
  </lst>
</requestHandler>
```

### Appends

`<lst name="appends">` で既に定義されているパラメータに追加するパラメータを定義することができる。
フィルタークエリなど、同じパラメータを複数回定義する必要がある場合に便利。
Solrにはクライアントがこれらのパラメータを上書きできる仕組みはないので、ここで定義されたパラメータは常に適用される。

```xml
<lst name="appends">
  <str name="fq">inStock:true</str>
</lst>
```

### Invariants

`<lst name="invariants">` ではクライアントがオーバーライドできないパラメータを定義できる（appendsと何が違う？）

```xml
<lst name="invariants">
  <str name="facet.field">cat</str>
  <str name="facet.field">manu_exact</str>
  <str name="facet.query">price:[* TO 500]</str>
  <str name="facet.query">price:[500 TO *]</str>
</lst>
```

## InitParams

2つ目の方法は `initParams` で別々のリクエストハンドラで共通して使うプロパティを定義すること。
ここで定義したパラメータをオーバーライドしたい場合は、各 `<requestHandler>` で同じパラメータを定義することで可能。
複数のリクエストハンドラで使用されるので `path` パラメータではワイルドカードを使うことができて、シングルアスタリスクは1階層深いネストされたパスを、ダブルアスタリスクはその下のすべてのネストされたパスを表す。

```xml
<initParams path="/update/**,/query,/select,/tvrh,/elevate,/spell">
  <lst name="defaults">
    <str name="df">_text_</str>
  </lst>
</initParams>
```

`path` を明示せず、`name` を設定することで、その `<initParams>` をパラメータセットとして `<requestHandler>` から参照できる。

```xml
<requestHandler name="/dump1" class="DumpRequestHandler" initParams="myParams"/>
```

## Paramsets and UseParams

3つ目の方法はRequest Parameters APIを使ってパラメータセットを定義すること。
頻繁にパラメータを変更することが予想される場合に便利で、`<requestHandler>` 要素のパラメータとして指定するか、`useParams` リクエストパラメータで指定することで使用できる。

```xml
<requestHandler name="/terms" class="solr.SearchHandler" useParams="myQueries">

...
</requestHandler>
```

```
http://localhost/solr/techproducts/select?useParams=myFacets,myQueries
```

# Update Request Processors

## URP Anatomy and Lifecycle

更新リクエストはUpdate Request Processorのチェーンによって処理される。
チェーンの各処理は[UpdateRequestProcessor](https://solr.apache.org/docs/9_4_0/core/org/apache/solr/update/processor/UpdateRequestProcessor.html)を継承したクラスで、それに対応した[UpdateRequestProcessorFactory](https://solr.apache.org/docs/9_4_0/core/org/apache/solr/update/processor/UpdateRequestProcessorFactory.html)を継承したクラスも必要。
更新リクエストプロセッサはリクエストごとに作成されるのでスレッドセーフである必要はないが、ファクトリクラスはスレッドセーフである必要がある。
また、更新リクエストプロセッサチェーンはSolrコアのロード中にロードされ、コアがアンロードされるまでキャッシュされる。

Solrが更新リクエストを受けると、使用するチェーンの検索、ファクトリーが新しいUpdateRequestProcessorのインスタンス作成、対応するUpdateCommandオブジェクトに解析という流れで進む。UpdateRequestProcessorはチェーン内の次のプラグインを呼び出す必要があるが、呼び出さずにチェーンを短縮させたり、例外を投げて処理を中断したりすることもできる。
