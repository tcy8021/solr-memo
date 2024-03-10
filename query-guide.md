# Common Query Parameters

| パラメータ名          | デフォルト値 | 説明                                                                                                                                                           |
| --------------------- | ------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| defType               | lucene       | メインリクエストパラメータ `q` を処理するパーサを指定                                                                                                          |
| sort                  | score desc   | 検索結果を並べる基準と順序を指定                                                                                                                               |
| start                 | 0            | 検索結果を表示する際のオフセット                                                                                                                               |
| rows                  | 10           | 一度に返す検索結果の数                                                                                                                                         |
| canCancel             |              | task management interfaceでキャンセルできるかどうか                                                                                                            |
| queryUUID             |              | `canCancel=true` のときにクエリを識別するためのID。指定されない場合は自動で割り当てられる。                                                                    |
| fq                    |              | スコアに影響を与えず、返されるドキュメントをフィルターするために使う。メインクエリとは別にキャッシュされるので高速化に便利。                                   |
| fl                    | *            | レスポンスに含めるフィールドを指定                                                                                                                             |
| debug                 |              | `query`, `timing`, `results`, `all` のどれかを指定し、それぞれに関する情報を返す                                                                               |
| explainOther          |              | メインクエリとは別のクエリを指定して、そのデバッグ情報を返す。メインクエリのデバッグ情報と比較して、結果が期待通りではない理由を特定するために使う。           |
| timeAllowed           |              | 検索完了までに許容される時間（ミリ秒）。時間切れになった場合、部分的な結果が返される。                                                                         |
| segmentTerminateEarly | fales        | trueの場合、コレクションの `mergePolicyFactory` によって、結果の現在のページの候補でないドキュメントをセグメントごとにスキップする                             |
| omitHeader            | false        | trueの場合、検索結果からヘッダを除外する                                                                                                                       |
| wt                    |              | レスポンスをフォーマットするためのレスポンスライター                                                                                                           |
| logParamsList         |              | ログに記録するリクエストパラメータのallow list                                                                                                                 |
| echoParams            | none         | レスポンスヘッダに含めるリクエストパラメータを制御する。`explicit`, `all`, `none` のいずれかを指定する。                                                       |
| minExactCount         |              | 正確にヒット件数をカウントする下限値。その後は上位N件に入るほど高いスコアを持たないドキュメントをスキップする。`score desc` で最初にソートする場合のみ使える。 |

# Dense Vector Search

## Index Time

### Dense Vector Field

スキーマで `DenseVectorField` を設定する方法。

```xml
<fieldType name="knn_vector" class="solr.DenseVectorField" vectorDimension="4" similarityFunction="cosine"/>
<field name="vector" type="knn_vector" indexed="true" stored="true"/>
```

## Query Time

k-最近傍クエリパーサ(`knn`)は、インデックス作成時に指定した類似度関数に従い、クエリに最も近いk個のドキュメントを見つける。
knnパーサはfqやリランクでも使用できる。

