---
title: next.js(vercel)でString.prototype.replaceAllをpolyfillする
emoji: 🩹
type: tech
topics:
  - nextjs
  - vercel
  - javascript
  - corejs
published: true
---

最近stringに生えた`replaceAll`はだいぶ便利だ。今まで`replace(/foo/g, "baz")`のようにしていたものを`replaceAll("foo", "baz")`などできる。

ただこれはnode.jsではv15からのみ入っていて、v14以下では利用出来ない。

[現状vercelはnode v14系](https://vercel.com/docs/runtimes)なので、利用するとランタイムエラーが起きる

そこで使うのを諦めるかpolyfillするかの選択肢になる。

調べると下記のようなissueが見つかった

* https://github.com/vercel/next.js/discussions/15715

要は`_app.js`に記載するのがお作法らしい。[^1]

例えば`core-js`をpolyfillとして利用するならこんな具合になる

```js
// _app.js
import 'core-js/features/string/replace-all' // 追加

function MyApp({ Component, pageProps }) {
  return <div>
        <Component {...pageProps} />
  </div>
}

export default MyApp
```

[^1]: [Custom Pollyfill](https://nextjs.org/docs/basic-features/supported-browsers-features#custom-polyfills)の項目にはサラッと書いてあるようだった