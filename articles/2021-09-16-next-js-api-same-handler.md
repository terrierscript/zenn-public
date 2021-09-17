---
title: next.jsで複数のAPIエンドポイントで同じhandlerを使う小技
emoji: ♊️
type: tech
topics: 
    - nextjs
    - react
    - typescript
published: true
---

next.jsで同じエンドポイントをハンドラとして使い回すとちょっと便利だった

## ユースケース1: URLを変更したい場合

APIのURLを変更したいが後方互換を保ちたいようなケースがあるだろう。
リダイレクトを設定したり、[Custom Server](https://nextjs.org/docs/advanced-features/custom-server)を利用しても良いが、ちょっとそこまでやりたくないがコピペもしたくないようなときに利用できる

例えばこんなAPIが`/api/greet`としてあったとする

```ts
// /api/greet.ts

import { NextApiHandler } from "next"

const handler: NextApiHandler = async (req, res) => {
  const { name } = req.query
  res.statusCode = 200
  res.json(`Hello ${name}`)
}
export default handler

```

これを`/api/[name]/greet`にしたくなったとする。

まず新API側をこのようにする

```ts
// /api/[name]/greet.ts
import { NextApiHandler } from "next"

// 利用可能なようにexportする
export const greetHandler: NextApiHandler = async (req, res) => {
  const { name } = req.query
  res.statusCode = 200
  res.json(`Hello ${name}`)
}

export default greetHandler
```

あとは古い方は下記のようにする

```ts
// /api/greet.ts

import { NextApiHandler } from "next"
import greetHandler from "./[name]/greet"

export default greetHandler
```

呼び出し時に警告メッセージを出したければこんな具合にしても良いだろう

```ts
const handler: NextApiHandler = (req, res) => {
  console.warn("Deprecated !")
  return greetHandler(req, res)
}

export default handler
```

## ユースケース2: クライアント向けのAPIを管理者向けに提供する

ユーザー向けに提供しているAPIをAdminツール向けに扱いたい場合も使えるだろう。
こちらは内部のロジックさえ共通化するほうが綺麗ではあるので、あまり活躍の出番はないかもしれないが、たまに便利なときがある

```ts
// /api/user/something.ts

import { NextApiHandler } from "next"

const handler: NextApiHandler = (req, res) => {
  const { name } = req.query
  res.statusCode = 200
  res.json(`Hello ${name}`)
}

export default userAuthorization(handler)
```

Admin向けに認証を変えるならこんな具合だろう。

```ts
// /api/admin/something.ts

import { NextApiHandler } from "next"
import { somethingHandler } from "../user/something"

export default adminAuthorization(somethingHandler)
```

### 認証したユーザーIDを利用するなら？
更に例えばアクセスしているユーザーの情報を利用するならこんな具合だろう

```ts
// /api/user/something.ts
import { NextApiHandler, NextApiRequest, NextApiResponse } from "next"

type UserAuthorizedNextApiHandler<T = any> = (req: NextApiRequest, res: NextApiResponse<T>, authorizedUserId: string) => void | Promise<void>

export const somethingHandler: UserAuthorizedNextApiHandler = (req, res, userId) => {
  const resource = getSomething(userId)
  res.statusCode = 200
  res.json(resource)
}

export default userAuthorization(somethingHandler)
```

Adminではクエリか何かでuserIdを受け取って返すことになるだろう

```ts
// /api/admin/something.ts
import { NextApiHandler } from "next"
import { somethingHandler } from "../user/something"

const handler: NextApiHandler = (req, res) => {
  const { userId } = req.query
  return somethingHandler(req, res, userId)
}

export default adminAuthorization(somethingHandler)
```

ただここまで来るならhandlerで行わず別途ロジックを再利用するほうが妥当なケースが多いかもしれない