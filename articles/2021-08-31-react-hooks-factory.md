---
title: React hooksでfactoryパターンっぽいことをしたいときは結局コンポーネントを分離したほうが良い
emoji: 🏭
type: tech
topics:
  - hooks
  - react
  - typescript
  - javascript
published: true
---

Reactでhooksを切り替えしたい時について考えてみた。

中盤は悪例を挟みます。悪例については利用しないことをおすすめします
結論だけ知りたい方向け -> [結論へ](#結論)

# 前提

React hooksで例えばパターンA、パターンBを切り替える、factoryパターンっぽいことをやりたくなることがある。

コンポーネントより外側で切り替えることが可能であれば[こちらの記事](https://dev.to/pietmichal/react-hooks-factories-48bi)などのようなことが可能だが、コンポーネントの内部で切り替えたいケースだとこのやり方だと処理出来ない

例えばこのようなカウンターを考えてみたい。

```ts

// Hooksは共通で下記の値を返す
export type CounterHooksResult = {
  value: number,
  onClick: () => void
}

// 単純なカウンター
export const usePlainCounter = (): CounterHooksResult => {
  const [state, setState] = useState(0)
  return {
    value: state,
    onClick: () => {
      setState(current => current + 1)
    }
  }
}

// ランダムにするカウンター
export const useRandomCounter = (): CounterHooksResult => {
  const [state, setState] = useState(0)
  return {
    value: state,
    onClick: () => {
      setState(Number((Math.random() * 100).toFixed(0)))
    }
  }
}

// フィボナッチ数列で増えるカウンター
export const useFibCounter = (): CounterHooksResult => {
  const [fib1, setFib1] = useState(1)
  const [fib2, setFib2] = useState(1)
  return {
    value: fib2,
    onClick: () => {
      const nextValue = fib1 + fib2
      setFib1(fib2)
      setFib2(nextValue)
    }
  }
}
```

これを例えば下記のように切り替えしてみたいというシーンとなる

```tsx
const Counter = ({ method }: { method: string }) => {
  const props = useこのカウンター用Hooksをカウンターによって切り替えたい()
  return <Stack>
    <Box>Value: {value}</Box>
    <Button onClick={onClick}>Click</Button>
  </Stack>
}
```

## とりあえずなんとかするなら？

### 🤨 全部呼び出す
かなり力技だが、例えば下記のように全部呼び出して切り替えるなら可能だろう。

```tsx

const useCallAllPattern = (method: string) => {
  const random = useRandomCounter()
  const fib = useFibCounter()
  const plain = usePlainCounter()
  switch (method) {
    case "random":
      return random
    case "fib":
      return fib
    default:
      return plain
  }
}

const Counter = ({ method }: { method: string }) => {
  const props = useCallAllPattern(method)
  return <Stack>
    <Box>Value: {value}</Box>
    <Button onClick={onClick}>Click</Button>
  </Stack>
}
```

無駄な呼び出しも多く、場合によってはパフォーマンスに問題がありそうで、あまり取りたくない手段だ。

# 駄目なパターン
ここから少し陥りそうな悪例パターンを紹介していく。
あくまで出来ないものとして読んでいただきたい

## ❌ hooksの内部で切り替える

下記のようにhooksの内部で呼び分けるのを考えたくなるかもしれないが、これはhooksルールに違反してしまう

```tsx
// ❌ ダメなパターン
const useCounter = (method: string) => {
  switch (method) {
    case "random":
      return useRandomCounter()
    case "fib":
      return useFibCounter() 
    default:
      return usePlainCounter()
  }
}

const Counter = ({ method }: { method: string }) => {
  const { value, onClick } = useCounter(method)
  return <Stack>
    <Box>Value: {value}</Box>
    <Button onClick={onClick}>Click</Button>
  </Stack>
}
```

useRandomCounterとusePlainCounterはhooksの呼び出し数が一緒なのでエラーは出ないが、useFibCounterなどに切り替えるとhooksのエラーとなる。

下記のようなエラーが出るだろう

```
Uncaught Error: Rendered more hooks than during the previous render.
```

hooksが同じ数の場合はエラーが出なかったりするが、いつコケるかわからない危険なコードなので一切やらないことをおすすめする

また、なんとなく「もしかしたら関数だけを返すなら行けるかも？」と思うかもしれないが、これも同様NGだ

```tsx
// ❌ ダメなパターン
const getFactoryCounter = (method: string) => {
  switch (method) {
    case "random":
      return useRandomCounter
    case "fib":
      return useFibCounter
    default:
      return usePlainCounter
  }
}

const Counter = ({ method }: { method: string }) => {
  const useGeneratedHooks = getFactoryCounter(method)
  const {value, onClick} = useGeneratedHooks()
  return <Stack>
    <Box>Value: {value}</Box>
    <Button onClick={onClick}>Click</Button>
  </Stack>
}

```

# ダメではないがおすすめしないパターン

## 🤨 hooksの内部で切り替えつつ、keyを設定して分離

上記の回避策として、`key`を与えることで一応回避は可能となる。
ただkeyの本来の使い方とは違い、保守性の観点からしても潜在的な危険性を抱えるので、これも個人的にはおすすめしない

```tsx
// おすすめしないパターン

const useCounter = (method: string) => {
  switch (method) {
    case "random":
      return useRandomCounter()
    case "fib":
      return useFibCounter()
    default:
      return usePlainCounter()
  }
}

const CounterInternal = ({ method }: { method: string }) => {
  const { value, onClick } = useCounter(method)
  return <Stack>
    <Box>Value: {value}</Box>
    <Button onClick={onClick}>Click</Button>
  </Stack>
}

const Counter = ({ method }: { method: string }) => {
  return <CounterInternal method={method} key={method} />
}
```


# 結論

ここからどうすればよいかについて記述していく。

## 🙆🏻‍♂️ コンポーネントで分離する

当たり前で面白くない結論ではあるが、やはりhooksでどうこうせずにコンポーネントの部分で解決するのが良いだろう。

少々記述量は多くなるが、コンポーネントを適切に共通化できればそれほど肥大化もしないはずだ

```tsx
// CounterComponentが共通コンポーネントとして定義
// hooksではなく結果値を受け取るコンポーネント側で共通化する
const CounterComponent = ({ value, onClick }: CounterHooksResult) => {
  return <Stack>
    <Box>Value: {value}</Box>
    <Button onClick={onClick}>Click</Button>
  </Stack>
}

// それぞれのHooksを利用するコンポーネントを定義する

const PlainCounterComponent = () => {
  const props = usePlainCounter()
  return <CounterComponent {...props} />
}

const RandomCounterComponent = () => {
  const props = useRandomCounter()
  return <CounterComponent {...props} />
}
const FibCounterComponent = () => {
  const props = useFibCounter()
  return <CounterComponent {...props} />
}

// コンポーネントとして切り替えを行う

const Counter = ({ method }: { method: string }) => {
  switch (method) {
    case "random":
      return <RandomCounterComponent />
    case "fib":
      return <FibCounterComponent />
    default:
      return <PlainCounterComponent />
  }
}

```


## 🙂 お好みで高階コンポーネント

個人的にはあまり読み味が好きではないが、記述量の多さが気になる場合は高階コンポーネントでComponentビルダーを作るのもありだろう。
それぞれの実装の分断を一切許したくないケースでも利用できるだろう。

```tsx

const buildCounterComponent = (useMethodHooks: () => CounterHooksResult) => {
  return function BuildCounter() {
    const props = useMethodHooks()
    return <CounterComponent {...props} />
  }
}

const PlainCounterComponent = buildCounterComponent(usePlainCounter)

const RandomCounterComponent = buildCounterComponent(useRandomCounter)

const FibCounterComponent = buildCounterComponent(useFibCounter)
```