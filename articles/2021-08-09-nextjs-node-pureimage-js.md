---
title: nextjsでnode-pureimageを使ったJSオンリーな動的画像生成
emoji: 🧚‍♂️
type: tech
topics:
  - nextjs
  - javascript
published: true
---

nextjsで動的画像生成をしたくなり、node-canvasだとnode以外の依存関係がちょっと困るケースがあったので、PureJSで実行できる[node-pureimage](https://github.com/joshmarinacci/node-pureimage/)でなんとかならないか色々やってみた


### 出来た結果

```ts
// /api/image/sample

import { NextApiHandler, NextApiResponse } from "next"

// @ts-ignore
import { make, encodePNGToStream } from "pureimage"

import getStream from 'get-stream'
import { PassThrough } from 'stream'

const generateSampleImage = async () => {
  // 画像生成
  const img = make(100, 100)
  const ctx = img.getContext("2d")

  ctx.fillStyle = 'black'
  ctx.fillRect(0, 0, 100, 100)
  ctx.beginPath()
  ctx.fillStyle = 'red'
  ctx.arc(50, 50, 40, Math.PI, 0)
  ctx.fill()
  ctx.closePath()

  // bufferに変換
  const stream = new PassThrough()
  await encodePNGToStream(img, stream)
  const buffer = await getStream.buffer(stream)
  return buffer
}

// レスポンス返却
const setResponse = (res: NextApiResponse<any>, buf: Buffer) => {
  res.setHeader('Content-Type', 'image/png')
  res.setHeader('Content-Length', buf.length)
  res.setHeader('Cache-Control', `public, max-age=86400`)
  res.send(Buffer.from(buf))
  res.status(200).end()
}

// ハンドラ
const handler: NextApiHandler = async (req, res) => {
  const buf = await generateSampleImage()
  setResponse(res, buf)
}

export default handler
```

node-pureimageはデフォルトでは画像書き込みするメソッドしかないのだが、nextからレスポンスとして返すにはbuffer化する必要がある。
今回は[get-stream](https://github.com/sindresorhus/get-stream)を利用したBufferに変換した。

また、テキストを書き込む場合は、あらかじめ`registerFont`する必要がある

```ts
// @ts-ignore
import { make,　registerFont, encodePNGToStream } from "pureimage"

// ....
// ....
const fontPath = path.join("./fonts", "MPLUSRounded1c-Light.ttf")
const fnt = registerFont(fontPath, "MPLUSRounded1c")
await fnt.loadSync()

const ctx = img.getContext("2d")

ctx.fillStyle = "blue"
ctx.font = "78pt 'MPLUSRounded1c'"
ctx.fillText("foo", 20, 500)
```

ただ、フォントについては、下記のような未調査の点が残った
* opentype.jsを利用しているようなのだが、otfファイルがうまく読み込めないっぽい
* フォントサイズが80ptを超えると、next.jsがタイムアウトしがちになる

そのため、OGPを生成したい要件であれば、[verce/og-image](https://github.com/vercel/og-image)に頼るのが良い可能性がある。
HTMLで生成し、puppeteerでスクリーンショットしており、なかなか迫力があるのでコードを見てみるのもお勧め。