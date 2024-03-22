---
title: BigQueryで日付を間引きして取得する
emoji: 🪒
type: tech
topics:
  - bigquery
  - sql
  - gcp
published: true
---

BigQueryで日付ごとなどでパーティションしているテーブルに対して、1年以上など長めにデータ見たいが全期間取ってしまうと読み込み量が多くなってしまうので困るので対策を考えた

## `GENERATE_DATE_ARRAY`を使って間引きする

指定期間の日付の配列を生成してくれる`GENERATE_DATE_ARRAY`という関数が存在する
* https://cloud.google.com/bigquery/docs/reference/standard-sql/array_functions#generate_date_array

第三引数は間隔を指定できる。例えば下記のような具合だ

```sql
SELECT GENERATE_DATE_ARRAY(
  DATE_SUB(CURRENT_DATE('Asia/Tokyo'), INTERVAL 7 day),  -- 開始
  CURRENT_DATE('Asia/Tokyo') -- 終了
, INTERVAL 2 day) -- 間隔をあける日付
```

```
// output
2024-03-15
2024-03-17
2024-03-19
2024-03-21
```

:::message
クエリを実行前に、BigQueryコンソール右上の「このクエリを実行すると、XXX KB が処理されます」の見込み量を確認して、想定通りに動いている確認をすることを推奨します。
:::

これで日付を一定期間サンプリングすれば、読み込み量を減らすことができる。

実際のクエリで利用する場合は下記のような感じ。`GENERATE_DATE_ARRAY`が配列で返ってくるので、これをIN句で分割しているフィールドに対して指定する

```sql
SELECT *
FROM sourced_data
WHERE DATE(_PARTITIONDATE) IN UNNEST(
    GENERATE_DATE_ARRAY(
        DATE_SUB(CURRENT_DATE('Asia/Tokyo'), INTERVAL 1 year), 
        CURRENT_DATE('Asia/Tokyo')
    , 
    INTERVAL 5 day
    ) 
)
```

`INTERVAL 5 day`で5日に一度になるので、1/5の読み込み量で済む。読み込み量を調整したければここの値を変更すれば良い。
逆に`WHERE DATE(_PARTITIONDATE) NOT IN UNNEST(...`とすれば4/5ぐらいに調整というのも出来る。
