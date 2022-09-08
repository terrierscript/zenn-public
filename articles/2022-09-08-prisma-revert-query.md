---
title: prismaのログからクエリを復元する
emoji: 🤸‍♀️
type: tech
topics:
  - prisma
  - typescript
published: true
---

prismaの結果にEXPLAINをかけたくなったときに、ログからクエリに復元したくなったので、その時のメモ

## ログを仕掛ける

まずログを出力するには、`prisma.$on`を利用する

```ts
const prisma = new PrismaClient({
    log: [{ level: "query", emit: "event" }]
  })

prisma.$on("query", async (event) => {
   // eventにログが出てくる
})

```

イベントはこのようなものが取得出来る

```js
{
    timestamp: 2022-09-08T13:21:27.104Z,
    query: 'SELECT `dogrun`.`Post`.`id`, `dogrun`.`Post`.`title`, `dogrun`.`Post`.`name`, `dogrun`.`Post`.`published`, `dogrun`.`Post`.`createdAt` FROM `dogrun`.`Post` WHERE (`dogrun`.`Post`.`title` = ? AND `dogrun`.`Post`.`published` = ? AND `dogrun`.`Post`.`createdAt` > ?) LIMIT ? OFFSET ? /* traceparent=00-00-00-00 */',
    params: '["xx,x",true,"2022-09-08 13:21:26.919 UTC",1,0]',
    duration: 27,
    target: 'quaint::connector::metrics'
  },
```

## クエリに戻す

### `event.param`を処理する。
`event.param`はJSON.serializeしたようなデータが入っているが、厳密にはパース出来ないケースもあるようになっている

```ts
JSON.parse('["x"x,x",true,"2022-09-08 13:28:12.201 UTC",1,0]')
=> Uncaught SyntaxError: Expected ',' or ']' after array element in JSON at position 4
```

そこで厳密にやりたい場合は、CSVとしてパースするとそこそこうまくいく。。今回はpapaparseというライブラリで回避策を取った。

```ts
import * as Papa from "papaparse"

export const parseParam = (param: string) => {
  const removeBlacket = param.replace(/^\[/, "").replace(/\]$/, "")
  if (removeBlacket === "") {
    return []
  }
  const parsed = Papa.parse(removeBlacket, {
    dynamicTyping: true,
  })

  return parsed.data[0]
}
```

パラメータがJSONとしてパース出来ない点についてはissueも上がっているので、長期的には解消するかもしれない
* https://github.com/prisma/prisma/issues/12441
* https://github.com/prisma/prisma/issues/6578

### `event.query`に戻す。

ここまででプレースホルダ化したクエリとパラメータが取得出来た。
今回はクエリにまで戻していく。

また、ここからは使い方によっては**SQLインジェクションが起きうる処理**となるので、利用の際は注意。

ここでは[sqlstring](https://www.npmjs.com/package/sqlstring)を利用する

```ts
import * as SqlString from "sqlstring"

const revertedQuery = SqlString.format(query, params)

```

これで実行されたSQLの取得が出来た。


## 参考

今回目的だったexplainをかける処理については
[prisma-mysql-explain](https://www.npmjs.com/package/prisma-mysql-explain)としてアップロードしている。

* ソースコード: https://github.com/terrierscript/prisma-mysql-explain

繰り返しになるが、今回の処理ははSQLインジェクションになりかねないような処理も含まれるので、あくまで**開発環境などでのパフォーマンス計測用**として利用するためのものであることを改めて記載したい。