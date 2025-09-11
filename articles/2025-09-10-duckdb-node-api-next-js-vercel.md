---
title: '@duckdb/node-api + next.jsをvercelで動かしたい'
emoji: 🦆
type: tech
topics:
  - duckdb
  - vercel
  - nextjs
  - typescript
published: true
---

`duck-db`をvercel上で動かしたい場合、ほとんど`@duckdb/wasm`で行われるケースが散見される。
ただクライアントでヘビーな処理をしたくないなどサーバーで行いたいケースがわりとある。

vercelで動かすにあたり、いくらかエラーにぶつかったのでその対処

# Module parse failed
これは特にvercel関係なく起きる。

`next.config.js`に下記設定をいれて解決
```js
module.exports = {
  serverExternalPackages: ["@duckdb/node-api"]
}
```


# GLIBC周りのビルドエラー
```
[Error: Failed to collect configuration for /version] {
  [cause]: [Error: /lib64/libstdc++.so.6: version `GLIBCXX_3.4.30' not found (required by /vercel/path0/node_modules/@duckdb/node-bindings-linux-x64/duckdb.node)] {
    code: 'ERR_DLOPEN_FAILED'
  }
}
```

`"@duckdb/node-api": ^1.3.4-alpha.27"`で起きる。

`1.3.2-alpha.25`までバージョンを落とすと回避できる

# Can't find the home directory

下記のようなサンプルクエリを実行したとする

```ts
export const querySample = async () => {

  const instance = await DuckDBInstance.create()
  const conn = await instance.connect()
  
  await conn.run(`INSTALL httpfs; `)
  await conn.run(`LOAD httpfs; `)
  
  const url = `https://example.com/some/items.csv`
  const response = await conn.run(`SELECT * FROM read_csv('${url}')`)
  return response.getRowObjectsJson()
}
```

これだと下記のようなエラーが発生する。

```
[Error: IO Error: Can't find the home directory at ''
Specify a home directory using the SET home_directory='/path/to/dir' option.] 
```

`SET home_directory`を`httpfs`の処理前に実行することでこれは解決

```ts
export const querySample = async () => {
 // ...
  await conn.run(`SET home_directory = '/tmp';`) // 追加
  await conn.run(`INSTALL httpfs; `)
  await conn.run(`LOAD httpfs; `)
  // ...

}
```

# Bad Request - this can be caused by the S3 region being set incorrectly.

~~これは暫定回避したが半未解決みたいな問題。~~

httpfsを経由して、S3を指定した場合、下記のようなエラーに見舞われた

```
Bad Request - this can be caused by the S3 region being set incorrectly.
```

~~Provided Regionは想定通りのもので一致していて、色々と設定を確認したが根本原因を見つけられなかった。
ローカルでは起きていないので、なんらかvercel上の噛み合わせの悪さがありそうに思える。~~

~~S3でのリクエストではなく、`aws-sdk/s3-request-presigner`で署名付きURLを発行してそれを利用して`read_csv('${presignedUrl}')`のようにしたところ一旦回避できたので今回はこれでよしとした~~

### 追記: SessionTokenを発行すると解決
どうやらSessionTokenを発行して設定すれば良いようだった

```sql
CREATE SECRET my_secret (
    TYPE s3,
    KEY_ID 'my_secret_key',
    SECRET 'my_secret_value',
    REGION 'my_region',
    SESSION_TOKEN 'STSで発行したsession token'
);
```
内部的な調査はしていないので理由はちょっと不明