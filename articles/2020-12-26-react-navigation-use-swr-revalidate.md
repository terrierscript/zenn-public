---
title: react-navigationで画面遷移したときにuseSWRをrevalidateしたいときの話
emoji: 🎢
type: tech
topics:
  - expo
  - reactnative
  - reactnavigation
  - swr
  - javascript
published: true
---

expoでuseSWRを利用した通信をした際、react-navigationのページ遷移時にキャッシュを飛ばそうとしてもどうもうまく行かないようだったのでメモ。
おそらくreact-queryなどwindow.focusが関与する処理だいたいに共通する方法と思われる

## 🙅だめだった例

```jsx
const SomeScreenComponent = () => {
  const {data, error} = useSWR("/api/list/some", fetcher, { 
    revalidateOnFocus: true, revalidateOnMount: true, refreshWhenHidden: true 
  })
  return <View>
    {/*  ... */}
  </View>
}
```

`revalidateOnMount`や`refreshWhenHidden`など試してみたが、react-navigationのイベントとしてはどれらも発動しないようだった。

また、useEffectのクリーンアップにすればいけるか？というのも考えたがこれもルーティングの仕方によってはうまくいかないことがあった

```jsx
const SomeScreenComponent = () => {
  const {data, error, revalidate} = useSWR("/api/list/some", fetcher, { 
    revalidateOnFocus: true, revalidateOnMount: true, refreshWhenHidden: true 
  })
  const handleClenup = () => {
    revalidate()
  }
  useEffect(() => {
    return () => {
      handleClenup()
    }
  },[])
  return <View>
    {/*  ... */}
  </View>
}
```

## 🙆 OKだった例：react-navigation側のuseIsFocused()と組み合わせる

そこで[`useIsFocused`](https://reactnavigation.org/docs/use-is-focused/)を組み合わせることにした。これであればナビゲーションがfocusを外れたと認識した場合にrevalidateしてくれるようになる

```jsx
const SomeScreenComponent = () => {
  const isFocused = useIsFocused() 
  const {data, error, revalidate} = useSWR("/api/list/some", fetcher, { 
    revalidateOnFocus: true, revalidateOnMount: true, refreshWhenHidden: true 
  })
  const handleClenup = () => {
    revalidate()
  }
  useEffect(() => {
    if(!isFocused){
      handleClenup()
    }
    return () => {
      handleClenup()
    }
  },[])
  return <View>
    {/*  ... */}
  </View>
}
```

これであれば思うように動いてくれるようだった。そもそもuseSWRを通さずuseEffectでfetchすれば良いというのもあるのだが、その場合でもisFocusedを利用する必要がありそうだ。

また、あまり深く検証してないが[useFocusEffect](https://reactnavigation.org/docs/use-focus-effect)を使ってもうまくいくはずだ。
useFocusEffectはuseCallbackと組み合わせる必要があるので注意だ。

```js
useFocusEffect(useCallback(() => {
  return () => { // cleanup関数として定義すると、blutのタイミングで実行される
    handleClenup()
  }
}))
```

### react-queryの場合は？
おそらく同様の処理が必要と思われる。`remove`を叩けば良さそうだ。

```js
const {data, error, remove} = useQuery("/api/list/some", fetcher)
const handleClenup = () => {
  remove()
}
```

## その他の解法

SWRもreact-queryもそれぞれ似たようなことをライブラリ化しているものを使うというのもあるだろう。
軽く中身を見た感じ、自分のユースケースでは過剰っぽかったので今回は避けた

* https://github.com/nandorojo/swr-react-native
* https://github.com/cherniavskii/react-query-navigation-native