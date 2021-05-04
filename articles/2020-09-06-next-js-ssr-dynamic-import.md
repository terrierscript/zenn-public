---
title: 'next.jsでSSRの問題が起きたときにはdynamicとssr:falseでお手頃SSR回避出来る'
export_from: "blog"
topics: 
  - nextjs
  - javascript
emoji: "🦜"
type: "tech" # tech: 技術記事 / idea: アイデア
published: true
terrier_raw_date: '2020-09-06T11:52:59.180Z'
terirer_raw_title:  'next.jsでSSRの問題が起きたときにはdynamic(import(...),{ssr:false})でお手頃SSR回避出来る'

---

next.jsは大変便利だが、SSRがデフォルト挙動になっているので、ちょこちょこ`window`が無いとかで怒られがちになる。
そういう場合は`dynamic`を利用すれば良い

https://nextjs.org/docs/advanced-features/dynamic-import#with-no-ssr

もともとdynamic importのための機能ではあるが、SSR回避にも使える。
だいたい上記公式ページにまとまっているが、ちょいちょい忘れるのでメモ

# 例示
ちょっと雑だが、こんな具合にwindowを利用しているようなコンポーネントがあったとする（SSR未対応な手頃な例示が見つからなかったので、行儀の良くない例示なのはご了承いただきたい）

```js
// components/ClientOnlyComponent.js
window.SOME_GLOBAL_ITEM = "FOO" 
export const ClientOnlyComponent = () => {
  return <div>{window.SOME_GLOBAL_ITEM}</div>
}
```

これをそのまま読み込むと

```
ReferenceError: window is not defined
```

のようなエラーが出る。そこで上記のdynamicを利用する。

```js
// pages/index.js

import dynamic from "next/dynamic"
const AvoidSSRComponent = dynamic(
  () => import('../components/ClientOnlyComponent')
    .then(modules =>  modules.ClientOnlyComponent), 
  {ssr: false}
)

export default function Home() {
  return (
    <div>
      <AvoidSSRComponent/>
    </div>
  )
}

```
`default export`にしていれば`() => import('../components/ClientOnlyComponent')`だけで済むが、今回はnamed exportにしたので`then(modules => ...)`としている。
第二引数に`{ssr:false}`をつけるのがポイントだ。

また、`components/ClientOnlyComponent.js`は`pages`ディレクトリ下に置いてしまうと、ページ対象としてコンパイルされて`build`時のみコケたりするので注意（`next dev`でやってるとコケないのでたまに気づかない）
