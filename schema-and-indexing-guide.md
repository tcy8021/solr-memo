- [Schema Elements](#schema-elements)
  - [Unique Key](#unique-key)
  - [Similarity](#similarity)
- [Field Type Definitions and Properties](#field-type-definitions-and-properties)
  - [Field Type Properties](#field-type-properties)
    - [General Properties](#general-properties)
- [Analyzers](#analyzers)
  - [Analysis for Multi-Term Expansion](#analysis-for-multi-term-expansion)
- [Tokenizers](#tokenizers)
  - [About Tokenizers](#about-tokenizers)
    - [When to Use a CharFilter vs. a TokenFilter](#when-to-use-a-charfilter-vs-a-tokenfilter)
- [Language Analysis](#language-analysis)
  - [Language-Specific Factories](#language-specific-factories)
    - [Japanese](#japanese)
- [Phonetic Matching](#phonetic-matching)
- [Reindexing](#reindexing)
  - [Changes that Require Reindex](#changes-that-require-reindex)
  - [Schema Changes](#schema-changes)
    - [Solrconfig Changes](#solrconfig-changes)
  - [Reindexing Strategies](#reindexing-strategies)

# Schema Elements

## Unique Key

スキーマのデフォルトと `copyfields` は `uniqueKey` として使用できない。また、`uniqueKey` の `fieldType` は解析されてはならず、いずれの `*pointField` 型であってもいけない。

## Similarity

Luceneのクラスで、ドキュメントをスコアリングするために使われる。

各コレクションはグローバルな類似度を1つ持つ。デフォルトでは暗黙的に `SchemaSimilarityFactory` を使用し、field typeごとに類似度を設定できるようになっている。

デフォルトの動作はスキーマのトップレベルの `<similarity>` 要素を宣言することで上書きできる。
- 引数無しで類似度クラスを直接指定
```xml
<similarity class="org.apache.lucene.search.similarities.BM25Similarity"/>
```

- 引数ありで類似度クラスのfactoryクラスを指定（引数が不要な場合はなくてもよい）
```xml
<similarity class="solr.DFRSimilarityFactory">
  <str name="basicModel">P</str>
  <str name="afterEffect">L</str>
  <str name="normalization">H2</str>
  <float name="c">7</float>
</similarity>
```

しかし、グローバルレベルで類似度を設定すると、field typeに `<similarity>` が宣言されている場合にエラーになる。これの例外として、明示的に `SchemaSimilarityFactory` を宣言して、`defaultSimFromFieldType` で類似度が設定されたfield typeを指定することで、明示的に類似度が宣言されていないfield typeのデフォルトの動作を指定することができる。

以下の例では、明示的に類似度が宣言されていないfield typeの `text_other` で `text_dfr` と同じ類似度が使用されることになる。
```xml
<similarity class="solr.SchemaSimilarityFactory">
  <str name="defaultSimFromFieldType">text_dfr</str>
</similarity>

<fieldType name="text_dfr" class="solr.TextField">
  <analyzer ... />
  <similarity class="solr.DFRSimilarityFactory">
    <str name="basicModel">I(F)</str>
    <str name="afterEffect">B</str>
    <str name="normalization">H3</str>
    <float name="mu">900</float>
  </similarity>
</fieldType>

<fieldType name="text_ib" class="solr.TextField">
  <analyzer ... />
  <similarity class="solr.IBSimilarityFactory">
    <str name="distribution">SPL</str>
    <str name="lambda">DF</str>
    <str name="normalization">H2</str>
  </similarity>
</fieldType>

<fieldType name="text_other" class="solr.TextField">
  <analyzer ... />
</fieldType>
```

`defaultSimFromFieldType` を使用せずに `SchemaSimilarityFactory` を宣言した場合、`BM25Similarity` が使用される。

# Field Type Definitions and Properties

## Field Type Properties

### General Properties

- `positionIncrementGap`

多値フィールドにいおて、複数の値間の距離を指定し、誤ったフレーズ一致を防ぐ。

この説明だとちゃんと理解できなかったので、現時点での解釈を少し詳細に書く（間違ってるかも）
そもそもposition incremetとは、テキスト内で単語の位置情報を示す値である。例えば、"This is a pen."というテキストの場合、This:1, is:2, a:3, pen:4のような感じになる。
ここで"game water"というフレーズ検索のクエリがあるとする。
位置インクリメントはgame:1, water:2である。
あるドキュメントのjanreという多値フィールドに["game", "water"]という値が入っているとき、`positionIncrementGap` が0の場合はヒットする可能性があるが、`positionIncrementGap` が100だとgameとwaterの位置は100離れている（game:1, water:101）と解釈され、ヒットする可能性が低くなる。

# Analyzers

## Analysis for Multi-Term Expansion

プレフィックスやワイルドカード、正規表現のようなクエリは自然言語ではないため、同義語やストップワードフィルタリングなどが機能しない。
multi-term expansionになるクエリを解析する場合、各ファクトリの `nomalize` メソッドが呼ばれ、意味をなさないファクトリを入力をそのまま返す。これはCharFilterとTokenFilterの両方に適用される。

ほとんどの使用例では上記の方法で問題ないが、multi-term expamsionになるクエリの制御を完全に分けることもできる。
```xml
<fieldType name="nametext" class="solr.TextField">
  <analyzer type="index">
    <tokenizer name="standard"/>
    <filter name="lowercase"/>
    <filter name="keepWord" words="keepwords.txt"/>
    <filter name="synonym" synonyms="syns.txt"/>
  </analyzer>
  <analyzer type="query">
    <tokenizer name="standard"/>
    <filter name="lowercase"/>
  </analyzer>
  <!-- No analysis at all when doing queries that involved Multi-Term expansion -->
  <analyzer type="multiterm">
    <tokenizer name="keyword" />
  </analyzer>
</fieldType>
```

# Tokenizers

## About Tokenizers

### When to Use a CharFilter vs. a TokenFilter

CharFilterとTokenFilterには関連した機能を持つもの（`MappingCharFilter` と `ASCIIFoldingFilter`）やほとんど同じ機能のもの（`PatternReplaceCharFilterFactory` と `PatternReplaceFilterFactory`）がある。
どちらを使うべきかは、使用するトークナイザや文字ストリームの前処理が必要かなどに依存する。

例えば `StandardTokenizer` を使っていて全体的な動作には満足しているが、特定の文字だけ動作をカスタマイズしたい場合、CharFilterを使ってトークナイズの前に文字の一部をマッピングしたりできる。

# Language Analysis

## Language-Specific Factories

### Japanese

- `JapaneseIterationMarkCharFilter` : 踊り字を展開した形に正規化する
- `JapaneseTokenizer` : 形態素解析でトークナイズし、各タームに品詞、基本形（見出し語）、読み、発音を注釈する
- `JapaneseBaseFormFilter` : 見出し語への置換を行う
- `JapanesePartOfSpeechStopFilter` : 設定された品詞のいずれかを持つタームを削除する
- `JapaneseKatakanaStemFilter` : 長音符号（U+30FC）で終わるカタカナのタームから長音記号を削除する
- `CJKWidthFilter` : 全角ASCIIを基本ラテン文字にして、半角カタカナを全角カタカナにする

# Phonetic Matching

発音が似ていて綴りが異なるものを一致させるための機能。
アルゴリズムの概要と比較は以下を参照
- [http://en.wikipedia.org/wiki/Phonetic_algorithm](http://en.wikipedia.org/wiki/Phonetic_algorithm)
- [http://ntz-develop.blogspot.com/2011/03/phonetic-algorithms.html](http://ntz-develop.blogspot.com/2011/03/phonetic-algorithms.html)

Beider-Morse Phonetic Matching (BMPM), Daitch-Mokotoff Soundex, Double Metaphoneは専用のfilterがある。
その他のアルゴリズムはPhonetic Filterのencoder引数で指定して使用する。

# Reindexing

Solrの設定変更にはインデックスの再作成が必要になるものがある。
インデックスの再作成とは、既存のインデックスを削除し、コーパス全体を取り込むために使用したプロセスを繰り返すこと。

## Changes that Require Reindex

以下の変更を行った場合はインデックスの再作成が必要

- スキーマの変更
  - フィールドの追加や削除
  - フィールドやフィールドタイプの定義変更
  - analysis chainsの変更
    - ただし、インデックスとクエリのチェーンが別々に定義されていて、クエリ側だけを修正する場合は再作成は不要
- solrconfigの変更
- Solrのアップグレード

## Schema Changes

ごく少数の例を除き、スキーマを変更するとインデックスの再作成が必要になる。
これは利用可能なオプションがインデックス作成中にのみ適用されるため。


より一般的な理由としては、インデックスはLuceneのもので、スキーマはSolr固有のものなので、インデックス作成後のスキーマの変更がインデックスに反映されないため。
スキーマはインデックスを作成する際にデータをどう解釈するかLuceneに指示するために使われる。

### Solrconfig Changes

solrconfigを変更する場合のインデックス再作成の基準は「インデックスに保存されるものを変更するものは、再インデックスが必要」である。
例えばluceneMatchVersionパラメータの変更、updateRequestProcessorChainの変更、codecFactoryパラメータなど。

## Reindexing Strategies
以下の2パターンがある

1. 既存のドキュメントをすべて削除して、インデックス処理を再実行（ベストな方法）
2. 別の名前で新しいコレクションを作成し、ドキュメントを入れ終わったらエイリアス機能を使って向き先を切り替える（SolrCloudのみ可能）
