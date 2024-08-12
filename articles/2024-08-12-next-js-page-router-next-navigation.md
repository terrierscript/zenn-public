---
title: next.jsのPage Routerでnext/navigationはどの程度使えるのか？
emoji: 🍈
type: tech
topics:
  - nextjs
  - react
  - approuter
published: true
---

next.jsをPages RouterからApp Routerへ移植する際、`next/router`の`useRouter`を置き換える必要が出てくる。
クエリやパラメータを取得するような処理をコンポーネントで書いていると、共通のコンポーネントでちょこちょこ困ることがあった

この置き換えにあたって、App Routerにエイヤで移さず

1. Pages Routerの上で利用しているコンポーネントの`next/router`の`useRouter`を`next/navigation`の`useParams`や`useSearchQuery`に置き換える
2. 置き換えが終わったらApp Routerへ移植する

という手順が踏めれば比較的安全に移行ができ、マイグレーションの手助けになりそうだと考えた

# `next/navigation`のhooksはPages Routerでも使って良いのか？

上記のようなマイグレーションパスについて明確な記載はないものの[useParams](https://nextjs.org/docs/app/api-reference/functions/use-params#returns)のドキュメントには

> If used in Pages Router, useParams will return null on the initial render and updates with properties following the rules above once the router is ready.

との記載があり、Pages Routerでの利用も想定されていると考えて良さそうな記述を見つけた

また、下記などのIssueを見ても互換性を可能な限り取ろうとしている
* [#54242](https://github.com/vercel/next.js/issues/54242)
* [#55280](https://github.com/vercel/next.js/pull/55280)
 
とのことなので基本的には先に置き換えて良いと考えて良さそうだ

## どのぐらい違いがあるのか？

ここからそれぞれどの程度違いがあるかの比較をしていく。

今回は

`sample/[p]/[[...q]].tsx`

というようなルーティングに対して、

`sample/a/b/c?d=e`

のようなページにアクセスした場合にどのような値になるか？というケースで比較していく

### パラメータ関連

```tsx
import { useRouter } from 'next/router'

const router = useRouter()

// パラメータ関連
console.log(router.query) 
// => {"d":"e","p":"a","q":["b","c"]} 

```

```tsx
import { useParams,  useSearchParams } from 'next/navigation'

const params = useParams()
const searchParams = useSearchParams()

// パラメータ関連
console.log(JSON.stringify(params))
// => {"p":"a","q":["b","c"]}
console.log(JSON.stringify(Object.fromEntries(searchParams ?? [])))
// => {"d":"e"}

```

* `useRouter`の`query`ではdynamic paramsもクエリパラメータもまとめて取得出来ていたが、`next/navigation`では`useParams`と`useSearchParams`は別々に取得する必要がある
  * `useSearchParams`は`URLSearchParams`の形で取得できるため、上記は表示のために`Object.fromEntries`でオブジェクトに変換している。
  * `useSearchParams`の返り値についての細かい部分は[ドキュメント](https://nextjs.org/docs/app/api-reference/functions/use-search-params#returns)に詳細があるので参照すると良い。

### パス関連

```tsx
import { useRouter } from 'next/router'

const router = useRouter()

console.log(router.pathname) 
// => /sample/[p]/[[...q]] 
console.log(router.asPath) 
// => /sample/a/b/c?d=e
```

```tsx
import { usePathname } from 'next/navigation'

const pathname = usePathname()

console.log(pathname)
// => /sample/a/b/c
```

パスについては完全一致するものはなさそうで、`/sample/[p]/[[...q]]`のような変換前の取得は難しかった。ナビゲーション関連の出し分けで使うことはまれにあったので、その場合は何らかの代替方法を工夫する必要がありそう。

### ナビゲーション関連

`push`, `back`等のナビゲーション関連は`next/navigation`の`useRouter`という同名のhooksに一通り存在しているので、これらを利用すれば可能。

ただしドキュメントにはPages Routerでの利用については記載がないので、少々注意が必要かもしれない。

### その他無さそうな機能

下記のような部分は存在しなさそうだった
* isReady、isPreview等の状態系
* locales関連

## まとめ
* 簡易なパラメータ取得系であればPage Router上で`next/navigation`を先回りして利用しておくことは現実的に可能そう
* 細かい動作については一致しないので、複雑な処理や特殊なについてはこの方法は慎重になる必要がある