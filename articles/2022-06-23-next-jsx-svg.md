---
title: next.jsでJSX記述したSVGを動的に返したい
emoji: 🤙
type: tech
topics:
  - nextjs
  - react
  - jsx
  - javascript
  - svg
published: true
---

next.jsでJSXで記述したSVGを返したくなったときのメモ

### SVGをJSXで記述

まず返したいSVGをJSXとして記述。

```tsx
// SvgExample.tsx
import { FC } from "react"

export const SvgExample: FC<{ color: string }> = ({ color }) => {
  return <svg
    version="1.1"
    xmlns="http://www.w3.org/2000/svg"
  // xmlns:xlink="http://www.w3.org/1999/xlink"
  >
    <rect x="0" y="0" width="100" height="100" fill="red" />
    <rect x="20" y="40" width="100" height="100" fill="blue" />
    <rect x="30" y="20" width="100" height="100" fill={color} />
  </svg>
}
```

せっかくJSXなので強みを活かすためにpropsを受け取るようにもしておく

### APIとして返す

next.jsでHTML以外を返したいとすると、現状では`/api`以下のハンドラとして返す必要がある。

SVGを返すのに`renderToString`を利用する

```tsx
//  /api/svg.tsx
import { NextApiHandler } from "next"
import { createElement } from "react"
import * as ReactDOMServer from 'react-dom/server'
import { SvgExample } from "../../lib/SvgExample"

const handler: NextApiHandler = async (req, res) => {
  const color = req.query.color
  const domSvg = ReactDOMServer.renderToString(
    <SvgExample color={color} />
  )
  res
    .status(200)
    .setHeader("Content-Type", "image/svg+xml")
    .send(domSvg)
}

export default handler

```
SVGを文字列として取り出されればあとは`Content-Type`をSVGになるように設定して返せば良い

`/api`でJSXを直接書くのが気持ち悪い場合は下記のように`createElement`を使うのも良いだろう

```js
  const domSvg = ReactDOMServer.renderToString(
    createElement(SvgExample, { color: "green" })
  )
```