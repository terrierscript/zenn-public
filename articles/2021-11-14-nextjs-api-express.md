---
title: nextjsのAPIでExpress呼び出す
emoji: 🚄
type: idea
topics:
  - nextjs
  - express
  - javascript
published: true
---

nextからexpressのインスタンスを噛ませて呼び出せる小ネタを見つけたのでメモ。

ほとんど使うこと無いと思われるが、何らかの理由で外部のアプリケーションを呼び出したいケースはありそう

### やり方

すべてのルーティングを受けたいので、`/api/express/[[...props]].ts`などcatch allなルーティングにする。


あとは`express`でインスタンスを生成し、`handler`で受け取る`req`と`res`をそのまま渡す。

```ts
// /api/express/[[...props]].ts
import { NextApiHandler } from "next"
import express from "express"

const app = express()
app.get('/api/express', function (req, res) {
  return res.send('root')
})

app.get('/api/express/greet', function (req, res) {
  return res.send('hello world')
})

const handler: NextApiHandler = async (req, res) => {
  app(req, res)
}

export default handler
```

今回はわかりやすく`express`でやってみたが、node標準のhttp APIなど、RequestがIncomingMessageでResponseがOutgoingMessageであるものならある程度使えるはずだ。

ちなみにこれはnext@12の`_middleware`で出来ないか？みたいなことを考えてやっていたが、そちらは[Edge Runtime](https://nextjs.org/docs/api-reference/edge-runtime)というサンドボックスな環境で動いており、`fs`等が使えないので動かすことは出来ない模様だった。