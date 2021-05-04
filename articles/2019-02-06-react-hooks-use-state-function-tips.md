---
title: react-hooksのuseStateでfunctionを管理させたい場合のTips
emoji: 🎍
type: tech
topics:
  - reacthooks
  - react
  - javascript
published: true
---

:::message
この記事は下記記事の移植です（記述日:2019/02/06)
https://qiita.com/terrierscript/items/6a8acbc7d1ce6521f879
:::

react-hooksの`useState`はこれまでのstateのように値を保持してくれる重要な関数だ。

例えば単純なカウンターならこんな具合になるだろう

```jsx
const useCounter = () => {
  const [count, setCounter] = useState(0)
  return { count, setCounter }
}
export const MyApp = () => {
  const { count, setCounter } = useCounter()
  return (
    <div>
      <div>{count}</div>
  　　  <button onClick={() => setCounter(count + 1)}>+</button>
    </div>
  )
}
```

この`useState`に関数を保持させたい場合、ちょっと注意が必要になる。

例えばこんな風に書くと、意図しない挙動になるだろう。

```jsx
const initialHelloFn = () => {
  console.log("initial")
}
const useCounter = () => {
  const [count, setCounter] = useState(0)
  const [helloFn, setHelloFn] = useState(initialHelloFn)
  // このnewFnが何度も呼び出される
  const newFn = useCallback(() => {
    console.log("hello!", count)
  }, [count])
  useEffect(() => {
    setHelloFn(newFn)
  }, [newFn, setHelloFn])
  console.log(helloFn)
  return {
    count,
    setCounter,
    helloFn
  }
}

export const MyApp = () => {
  const { count, setCounter, helloFn } = useCounter()
  return (
    <div>
      <div>{count}</div>
      <button onClick={() => setCounter(count + 1)}>+</button>
      <button onClick={() => helloFn()}>hello</button>
    </div>
  )
}
```

おそらくこのようにすると、`hello! 0`のような出力が大量に出てしまうだろう


# 正しく動かす場合はどうするか？

`{fn: 管理したい関数}` のようにobjectで管理する。

```js
const useCounter = () => {
  const [count, setCounter] = useState(0)
  // {fn: Function}などobjectの形にラップする
  const [helloFn, setHelloFn] = useState({ fn: initialHelloFn })
  const newFn = useCallback(() => {
    console.log("hello!", count)
  }, [count])

  useEffect(() => {
    setHelloFn({ fn: newFn })
  }, [newFn, setHelloFn])

  return {
    count,
    setCounter,
    helloFn: helloFn.fn
  }
}
```

# なぜこうする必要があるか？
結論から言えば`useState`から返ってくる`setFoo`のようなハンドラーはFunctional Updateに対応しているため、単純に関数を渡してしまうとFunctional Updateとして処理されてしまうからになる。

https://reactjs.org/docs/hooks-reference.html#functional-updates

Functional Update自体は便利で、下記のように、現在の値を引き継がずに関数だけで利用することができる

```jsx
<button onClick={() => setCount(prevCount => prevCount + 1)}>+</button>
```

これはhooksで新しく入ったわけではなく、`React.Component.setState`にも同様の機能が存在していたものだ
https://reactjs.org/docs/react-component.html#setstate

ただComponentの場合はstate自体がobjectなのでほとんどこのような引っかかり方をすることは無かった。`useState`が単純な値を格納するものとして利用できてる分、この点は気をつけるべき部分だろう
