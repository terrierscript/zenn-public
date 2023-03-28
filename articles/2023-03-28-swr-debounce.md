---
title: SWRでdebounceしつつ、値が変更されたら即座に以前の結果を表示しないようにする
emoji: 🏓
type: tech
topics:
  - swr
  - typescript
  - react
published: true
---

SWRにおいて、下記の要件を満たすようなことをしたいとき少し困った

* 入力についてdebounceやthrottleで抑止したい
* 入力値が変わったら、前回の値は表示しないようにしたい

前回の値が表示されないようにしたいケースはそれほど多くないが、例えば金額を取り扱うような場合で、混乱を起こしてしまうようなケースを想像してもらうとわかりやすいだろう。

今回は入力値のフィボナッチ数が支払額になるような入力フィールドを例にしてみる

# 事前準備：素直に作るとき（Debounceしない）

`/api/fib`というフィボナッチ数を返すAPIを素直理に利用するとこのような具合になる。`apiCallCount`については後ほどdebounceが正しくされているかを見るためのデバッグ用の値である
```ts
const useFibonacci = (value: number) => {
  const [apiCallCount, setApiCallCount] = useState(0)
  const result = useSWR(["fibonacch", value], async () => {
    const response = await fetch(`/api/fib?value=${value}`)
    const json = await response.json()
    setApiCallCount(num => num + 1)
    return json.fib
  }, {
    keepPreviousData: false
  })
  return {
    result,
    apiCallCount
  }
}
```
また、結果表示コンポーネントは下記のようにしておく

```tsx
const FibonacchResult: FC<{ value: number, isLoading: boolean, fibResult: number, apiCallCount: number }> = ({ value, isLoading, fibResult, apiCallCount }) => {
  if (isLoading) {
    return <div>
      <div>fib({value}) = Loading...</div>
      <div>API Call: {apiCallCount}</div>
    </div>
  }

  return <div>
    <div>fib({value}) = {fibResult}円</div>
    <div>API Call: {apiCallCount}</div>
  </div>
}
```
結果に「円」をつけてるのはユースケースを想像しやすいようにするためだけに行っている。

あとは下記のように組み合わせる。

```tsx
const NoDebounceFibonacch: FC<{ value: number }> = ({ value }) => {
  const { result, apiCallCount } = useFibonacci(value)
  return <FibonacchResult value={value}
    isLoading={result.isLoading}
    fibResult={result.data} apiCallCount={apiCallCount}
  />
}
```

下記のようにdebounceされずに処理されている様子がわかる
![](https://storage.googleapis.com/zenn-user-upload/8c62e85fc6a6-20230328.gif)

# Debounce
ここからdebounceを組み合わせていく。
debounce処理については[usehooks-ts](https://usehooks-ts.com/react-hook/use-debounce)のものを利用する

## 1. 普通にDebounceする

debounceを普通に組み合わせると下記のようになる。
```ts
const useFibonacciWithDebounce1 = (value: number) => {
  const [apiCallCount, setApiCallCount] = useState(0)
  const debounceValue = useDebounce(value)
  const result = useSWR(["fibonacch_debounce_1", debounceValue], async () => {
    const response = await fetch(`/api/fib?value=${debounceValue}`)
    const json = await response.json()
    setApiCallCount(num => num + 1)
    return json.fib
  }, {
    keepPreviousData: false
  })
  return {
    result,
    apiCallCount
  }
}
```
ただ、この状態だと`keepPreviousData:false`を設定していても`debounceValue`自体が変化するまでの間、前の結果が残ってしまうため今回の要件だと不足してしまう。

![](https://storage.googleapis.com/zenn-user-upload/53374bb8c05a-20230328.gif)

検索結果のサジェストなどのユースケースならここまでで十分な場合が多いが、今回はこの部分を対処したいので、もう少し調整が必要になる

## 2. Debounceされたら以前のデータを表示しないようにする

値が変化した場合に即座に値を消すために、debounce中であればキーをnullにする

```ts
const useFibonacciWithDebounce2 = (value: number) => {
  const [apiCallCount, setApiCallCount] = useState(0)
  const debounceValue = useDebounce(value)
  const isDebouncing = debounceValue !== value
  const result = useSWR(isDebouncing ? null : ["fibonacch_debounce_2", debounceValue],
    async () => {
      const response = await fetch(`/api/fib?value=${debounceValue}`)
      const json = await response.json()
      setApiCallCount(num => num + 1)
      return json.fib
    }, {
    keepPreviousData: false
  })
  return {
    result,
    apiCallCount
  }
}
```

フラグで`null`を渡す手法については[こちらのissue](https://github.com/vercel/swr/issues/1082#issuecomment-931913440)などでも類似の例が取り上げられている。

ただこの場合、データが無いかつローディングが始まってない(`data:undefind`かつ`isLoading:false`)という状態が一瞬発生してしまうようだった。

![](https://storage.googleapis.com/zenn-user-upload/7069d958932a-20230328.gif)

詳細は調べきれなかったが、内部処理中に何らかのラグが起きていると考えられる。

## 3. Debounceしつつ、Loadingの表示を意図どおりにする

一瞬何も表示されない部分については、`data`と`isLoading`を組み合わせた状態から新たにloading状態を作れば解決可能そうだった。

```tsx
const WithDebounceFibonacch3: FC<{ value: number }> = ({ value }) => {
  const { result, apiCallCount } = useFibonacciWithDebounce2(value)
  const isLoading = result.isLoading || result.data === undefined
  return <FibonacchResult value={value}
    isLoading={isLoading}
    fibResult={result.data} apiCallCount={apiCallCount}
  />
}
```

![](https://storage.googleapis.com/zenn-user-upload/d82bab546ecd-20230328.gif)

hooksの内部まとめたり、`useSWR`の結果を上書きする形でインターフェースを揃えるようなやり方も存在するが、今回はそこまではしないこととした

## まとめ
最後にここまで作ったものを同時に並べるとこのようになる。

![](https://storage.googleapis.com/zenn-user-upload/0756e19d0135-20230328.gif)