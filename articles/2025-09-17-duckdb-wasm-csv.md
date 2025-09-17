---
title: duckdb-wasmでCSVを扱うときに、それなりのtype-safeで安寧を得るまで
emoji: 🧆
type: tech
topics:
  - duckdb
  - typescript
  - zod
published: true
---

DuckDB-WASMをブラウザで使ってCSVデータを扱う際、色々と困りごとに当たったので、それをまとめていく。

今回クエリビルダーやORMでDuckDBで使えそうな適切なものがなかったので、そのあたりナシの生SQLで扱いつつ、Zodでそれなりに型もついてる状態をゴールとする

ちなみに今回は[駅データ.jp](https://www.ekidata.jp/)にて配布されているCSVをサンプルに取り扱ったので、サンプルコードもそれに沿った形となっている

# 1. CSV読み込みは`all_varchar=true`してしまう

DuckDBで`read_csv`でCSVを読み込む際、通常はそれぞれのフィールドを読み取って自動で型を決定してくれる
一見便利そうに見えるが、これを利用した場合

- 数値系がBigIntになり、JSON.stringfiyがコケるなど不便な引っかかりが多い
- 12:34 のような時間フォーマットが意図しないepochtime化されて、ややこしい
- float型はBufferになるケースがあり、地味にこまる

と色々と困ることが多かった。

read_csvのオプションとして、[`all_varchar=true`](https://duckdb.org/docs/1.3/data/csv/overview#parameters)を指定すると、すべての値を文字列として扱うようになるので、これに頼ることにした

```sql
CREATE OR REPLACE TABLE ${table} AS SELECT * FROM read_csv('${url}', all_varchar=true);
```

`auto_type_candidates`のようなパラメータや、`[castBigIntToDouble](https://shell.duckdb.org/docs/interfaces/index.DuckDBQueryConfig.html)`のような設定も試したのだが、どれもしっくりくるパターンが見つからず、一旦全部文字列化したほうが幸せになれそうという結論になった

### 数値でほしいときはCastする

`all_varchar=true`の副作用としてすべて文字列として扱われるので、例えばソートなどで困るケースはある。そのような場合は `foo::DOUBLE`のように[キャスト](https://duckdb.org/docs/stable/sql/expressions/cast)を明示的にすることで対応が必要

```sql
SELECT station,
FROM station
ORDER BY station.open_ymd::DATE
```

### (余談) SELCT * FROM read_csv で読み出すより CREATE TABLE　で作ったほうがいい時がある

ちょっと脇道に逸れるが、Duck DBでは`SELECT * FROM read_csv(...)`の直接読み出しも可能。
ただ`CREATE TABLE`でテーブルを作るのはそれはそれで利点があると感じた。

**メリット**
- console.logにCREATE TABLE文を出力しておけば、`duckdb -ui`でのデバッグ時にコピペで環境再現できる。S3のsigned URLなど、一時的なクエリだと特に利点大きい
- テーブル名を先に決めておけるのでクエリの使い回しが楽

**デメリット**
- 使わないテーブルのセットアップも含まれるので、テーブルが多いと困る


# 2. 結果をApache ArrowTable形式からZodで型付けした状態にする

DuckDBの結果はApache Arrow形式で返ってくる。これをJavaScriptで扱いやすい形に変換する必要がある。

Apache Arrowの中身はProxyが含まれていたりでそのままだとzodのパースを通せない。

ちょっとトリッキーだが`JSON.parse(JSON.stringify())`で変換できる。ただしbigintなど含まれるケースがあるので、この点はreplacerで対応。

```ts
import * as arrow from 'apache-arrow'

export const parseArrowTable = (record: arrow.Table) => {
  return record.toArray().map(t => {
    return JSON.parse(
      JSON.stringify(t, (_, value) => {
        if (typeof value === "bigint") {
          return Number(value)
        }
        return value
      })
    )
  })
}
```


# 3. SELECT文は`*`にせずテーブル名を書くとそのまま構造化出来て便利

MySQLあたりに慣れていると

```sql
SELECT A.* FROM foo AS A
```
のようにSELECTを`*`を利用する形で書きがちだが、

```sql
SELECT A FROM foo AS A
```

とテーブルごと指定することで、[structの形で返ってくる](https://duckdb.org/docs/stable/sql/query_syntax/from)。後述するがこれは型をつけるのに都合が良い

```sql
SELECT A,B FROM foo AS A JOIN baz AS B ON A.id = B.a_id
```
JOINの場合はこのような感じ。`[{A: {...}, B: {...}}...]` のような形で帰ってくる

実例で見ると

```sql
SELECT company.*, line.* FROM company 
JOIN line ON line.company_cd = company.company_cd 
LIMIT 1
```
とした場合

```

[
  {
    "company_cd": "3",
    "rr_cd": "11",
    "company_name": "JR東海",
    "company_name_k": "ジェイアールトウカイ",
    "company_name_h": "東海旅客鉄道株式会社",
    "company_name_r": "JR東海",
    "company_url": "http://jr-central.co.jp/",
    "company_type": "1",
    "e_status": "1",
    "e_sort": "1001",
    "line_cd": "1001",
    "line_name": "中央新幹線",
    "line_name_k": "チュウオウシンカンセン",
    ...
  }
]
``` 
このようにフラット化するところが

```sql
SELECT company, line FROM company JOIN line ON line.company_cd = company.company_cd LIMIT 1
```
とすることで

```
[
  {
    "company": {
      "company_cd": "3",
      "rr_cd": "11",
      "company_name": "JR東海",
      "company_name_k": "ジェイアールトウカイ",
      "company_name_h": "東海旅客鉄道株式会社",
      ...
    },
    "line": {
      "line_cd": "1001",
      "company_cd": "3",
      "line_name": "中央新幹線",
      "line_name_k": "チュウオウシンカンセン",
      "line_name_h": "中央新幹線",
      ...
    }
  }
]
```
のように構造化した状態になってくれる

# 4. Zodで型安全性にする

2の段階でApache Arrow Tableの形式を剥がしたので、あとはzodで通せばある程度型安全な状態にできる。`Record<string,string>`のままでも構わないといえば構わないが、やっぱりキー名ぐらい決まっていると幸せ具合が違う

`all_varchar=true`で読み込んでいるので、全て`z.string().nullable()`にしておけばざっくりと型定義可能になる。
（今回zodを使っているが、valibotなりarkTypeなり、好みに合わせて使うと良い）

```ts
export const StationSchema = z.record(z.enum([
  "station_cd", "station_g_cd", "station_name", "station_name_k", 
  "station_name_r", "line_cd", "pref_cd", "post", "address", 
  "lon", "lat", "open_ymd", "close_ymd", "e_status", "e_sort"
] as const), z.string().nullable())
```

キー部分については
```ts
const sample = await conn.query(`SELECT * FROM ${table} LIMIT 1;`)
console.log(r.toArray().map(t => Object.keys(t.toJSON()) ))

```
みたいな形で抜き出すことが可能。

JOINした結果も、struct化して受け取っているのでだいぶ簡素に受け取ることができる

```ts
// SELECT station, line, company 
// FROM station 
//   JOIN company ON station.company_cd = company.company_cd 
//   JOIN line ON line.line_cd = station.line_cd LIMIT 1

const resultSchema = z.object({
  station: StationSchema,
  company: CompanySchema,
  line: LineSchema
})
```
