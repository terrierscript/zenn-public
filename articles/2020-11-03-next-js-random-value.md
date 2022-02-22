---
title: next.jsで乱数を使う時にハマったメモ
emoji: 🕸
type: tech
topics:
  - nextjs
  - javascript
  - typescript
published: true
---

next.jsで「更新するたびにランダムにUUIDが生成される」みたいなページを作っていた時にハマったメモ。

# 現象

例えばこんな具合のコードを書いた場合

```jsx
import { v4 } from "uuid" // uuid.v4()はUUIDをランダム生成する関数

export default function RandomComponent ()  {
  const [uuid, setUuid] = useState(v4())
  return <div>
    uuid1: {uuid1} 
  </div>
}
```

この場合、SSRで一度UUIDが生成され、その後クライアントで再度UUIDが発行される。
そのためミスマッチが発生してこんなエラーが出る。

```
Warning: Text content did not match. 
Server: "c5d1f100-b9a1-4ee5-9f53-9a659265872e" 
Client: "14a70ba1-546d-49e9-9f8b-0f2938969f87"
```

ただエラーは出つつ、クライアント側では二回目に発行されたUUIDが出力される。

ここまでは想定内だが、この乱数をプロパティとして扱うとちょっと想定外なことが起きる。

```jsx
import { v4 } from "uuid" // uuid.v4()はUUIDをランダム生成する関数

export default function RandomComponent ()  {
  const [uuid, setUuid] = useState(v4())
  return <div>
    <a href={uuid1}>　{/* ①この値はレンダリング後にも更新されない */}
      uuid: {uuid}　{/* ②この値はレンダリング後にも更新される */}
    </a>
  </div>
}
```

そしてこれをdevではなくサーバーで実行すると、②側のtextノード部分はリロードのたびに毎回ランダムに変更されるのだが、①のDOMのプロパティとして設定した方はリロードしても更新されなくなる。

おそらく[Automatic Static Optimization](https://nextjs.org/docs/advanced-features/automatic-static-optimization)が効いてしまっているものと思われる
[^1]: どうもpropertiesの場合にこのような挙動があるようだ（closeはされてるが解決されてない模様）
https://github.com/vercel/next.js/issues/7322

また、そのため下記のように`getServerSideProps`がある場合は無効化され、リロードするたびに更新される。

```js
// 下記のように該当コンポーネントを利用するページでgetServerSidePropsがあると無効になる
export const getServerSideProps = () => ( {
  props: {}
})
```

そもそもWarningが出ているのでものではあるのだが、それにしてもプロパティだけが更新されないというのは若干不自然で気付きづらい部分でもあるので解決法をまとめる

# 解決法

## 1. `next/dynamic`を利用する解決法

おそらく一番正攻法かつコストが少ないのがこの方法。
`{ssr: false}`でdynamic importすることでSSRしなくなるので解決される。
ただし、クライアントに処理を移譲する方法となるためSSRの恩恵は受けられなくなる

```jsx
const DynamicComponent = dynamic(() => import("../components/dynamic"), { ssr: false })

export default function Page ()  {
  return <DynamicComponent />
}
```

## 2. getServerSidePropsを利用した解決法 

もし乱数をサーバー側で決定するだけで良いなら、getServerSidePropsを利用してもよいだろう。

```jsx
export default function Page({ serverSideValue }) {
  return <div>
    <a href={`${serverSideValue}`}>
      serverSideValue: {serverSideValue}
    </a>
  </div>
}

export const getServerSideProps = () => {
  return {
    props: {
      serverSideValue: v4()
    }
  }
}
```

## 3. useEffectを利用した解決法 (使い所を選ぶ)

dynamic importの劣化版みたいなところがあるが、useEffectを利用した解決方法もある

```jsx
const EffectComponent = () => {
  const [uuid, setUuid] = useState(null)
  useEffect(() => {
    setUuid(v4())
  }, [])
  
  if (!uuid) {
    return null
  }
  return <div>
    <a href={uuid}>
      Effect Component: {uuid}
    </a>
  </div>
}
```

これであればSSR時には`null`が返却されて、クライアント上でuseEffectによってUUIDが決定される。
