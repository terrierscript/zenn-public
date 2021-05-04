---
title: next.jsのAPIを利用して画像proxyのAPIを作る
emoji: 🏙
type: tech
topics:
  - nextjs
  - javascript
published: true
---


:::message warn
この記事で紹介しているものはnext.jsで利用できる手法ではあるものの、利用するサービスによっては制限やポリシーが存在する場合があります。
Vercelで利用する場合は[Fair Use Policy](https://vercel.com/docs/platform/fair-use-policy)を確認することをお勧めします。
:::

CORSの回避などでちょこちょこ欲しくなるproxy機能。
[netlifyの場合にはproxyでできた](https://docs.netlify.com/routing/redirects/rewrites-proxies/#limitations)が、next.jsをその他の環境で使う場合はAPI機能部分を使うのが良さそうだった。

```js
// /api/img.js
import axios from "axios"

export default async (req, res) => {
  try {
    const r = await axios.get(req.query.url, {
      responseType: "arraybuffer",
    })
    res.end(r.data)
  } catch (e) {
    res.end(500)
  }
}
```

これだけで出来るのでお手軽。

例えば拡張してリサイズ機能を入れたりヘッダも引き継ぎたい場合はこんな具合になる

```js
import axios from "axios"
import sharp from "sharp"

export default async (req, res) => {
  try {
    const r = await axios.get(req.query.url, {
      responseType: "arraybuffer",
    })
    const buf = r.data
    // resize
    const item = await sharp(buf).resize(800).toBuffer()

    // headderを引継ぎ
    Object.entries(r.headers).map(([k, v]) => {
      res.setHeader(k, v)
    })
    res.end(item)
  } catch (e) {
    res.end(500)
  }
}
```

