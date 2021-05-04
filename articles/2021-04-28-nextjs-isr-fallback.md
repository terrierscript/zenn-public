---
title: nextjsのISRを使うときのfallback指定について理解するまでの話
emoji: 🍃
type: tech
topics:
  - nextjs
  - isr
  - javascript
  - typescript
published: true
---

next.jsのISRを使おうとして「なんか全然うまく行かない」ってなってたのがやっと理解出来たのでメモ

## 問題編

とりあえず見様見真似でISRはrevalidateとfallbackつければ良いんだな？とやってみたところ、どうもpropsが空Objectになってしまうようで悩んでいた。

例えば下記のような場合、エラーが起きる

```jsx
// pages/greeting/[name].js

const Page = (props) => {
  // ↓ここでエラー
  return <div>Hello {props.name.toUpperCase()}</div>
}

export const getStaticProps = async (req) => {
  return {
    props: {
      name: req.params.name
    },
    revalidate: 100
  }
}

export const getStaticPaths = async (req) => {
  return {
    paths: [],
    fallback: true
  }
}

export default Page
```

```
TypeError: Cannot read property 'toUpperCase' of undefined

   9 |   // }
  10 | 
> 11 |   return <div>Hello {props.name.toUpperCase()}</div>
     |                                ^
  12 | }
```

ここからよくよく調べたりドキュメントを読んだりしているとどうやらこれだけでは駄目らしいということがわかってきた。

# 解決編

* https://nextjs.org/docs/basic-features/data-fetching#fallback-true

まず`fallback: true`の挙動として文字通り「fallback向けページ」を生成する。
fallbackページというのがはっきりしなかったが、これが「`props`が空Object状態のページ」ということらしい。
このあとgetStaticPropsを呼び出し、そのJSONを当てはめた状態で再レンダリングするようだ。

TypeScriptで表せば、こういう違いになるだろう

```tsx
// 通常のSSRの場合
const Page : FC<{ foo: string, baz: string}> 

//`fallback:true`にした場合
const Page : FC<{ foo: string, baz: string} | {}> 
```

ここから解決策を見ていく

## 解決策A: fallback: "blocking"を指定する

* https://nextjs.org/docs/basic-features/data-fetching#fallback-blocking

これは簡素。`fallback: "blocking"`にすればよい。

```jsx

export const getStaticPaths: GetStaticPaths = async (req) => {
  return {
    paths: [],
    fallback: "blocking"
  }
}
```

必ず先にビルドが走るので、`props`が入っている状態になる

## 解決策B: `fallback: true`にしたまま`router.isFallback`の判定を入れる

* https://nextjs.org/docs/basic-features/data-fetching#fallback-pages

`fallback: true`でのパフォーマンスを維持したいなら、下記のように`router.isFallback`を利用する。

```jsx
import { useRouter } from "next/router"

const Page = (props) => {
  const router = useRouter()

  if (router.isFallback) {
    return <div>Loading...</div>
  }

  return <div>Hello {props.name.toUpperCase()}</div>
}
```

こうするとまず`isFallback=true`の状態で空オブジェクトになり、その後`getStaticProps`を取得する処理が走る。

## 備考：`next dev`ではISRの検証は出来ない

* https://nextjs.org/docs/basic-features/data-fetching#runs-on-every-request-in-development

> In development (next dev), getStaticProps will be called on every request.

とのことで、devモードでは毎回`getStaticProps`が呼び出される模様
