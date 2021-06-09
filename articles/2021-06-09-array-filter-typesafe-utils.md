---
title: 配列のfilter()の型対応にtypesafe-utilsを使う
emoji:🦦
type: tech
topics:
  - typescript
published: true
---

TypeScriptで配列の`filter`を使用する場合を考える

```ts
// itemsは (string|number)[]
const items = [1, 2, "foo", "baz"]
const stringItems: string[] = items.filter(item => typeof item === "string")
```


`stringItems`が`string[]`になってくれることを期待したいところだが、これは`(string|number)[]`のままだ。

これを解決するにはType Guardを使う必要がある

```ts
const items = [1, 2, "foo", "baz"]
const isString = (item): item is string => {
  return typeof item === "string"
}

const stringItems: string[] = items.filter(isString)
```

もしくはこう

```ts
const stringItems = items.filter((item): item is string => {
  return typeof item === "string"
})
```

また一方で思い切って`as`や`@ts-ignore`を使ってしまうというのも一つの考え方だろう

```ts
// @ts-ignore
const stringItems: string[] = items.filter(item => typeof item === "string")
```

```ts
const stringItems: string[] = items.filter(item => typeof item === "string") as string[]
```

### 課題

Type Guardは単なる宣言のため、誤った情報を返しうる。

```ts
const items = [1, 2, "foo", "baz"]
const isString = (item): item is string => {
  // 何らかの不幸な改修でこうなってもTypeScriptは検知しない
  return typeof item === "string" || item === "number"
}

const stringItems: string[] = items.filter(isString)
```

これを避けるにはTypeGuardの関数を入念にテストするなどしかないだろう。


### 解決策

そういう場合は[`typesafe-util`](https://github.com/ivanhofer/typesafe-utils)が便利な選択肢になりうるだろう。

あまりスター数も現時点で多くないライブラリではあるものの、`isString`,`isTrue`,`isTruthy`,`isNumber`など一通りのものが揃っており型があり[テストも一通り行われている](https://github.com/ivanhofer/typesafe-utils/blob/main/src/isString/isString.test.ts)。

例えば先程実装した`isString`であればこんな具合だ。

```ts
import { isString } from 'typesafe-utils'

const items = [1, 2, "foo", "baz"]
const stringItems = items.filter(isString)
```

もちろんあまり複雑なことをやるなら自前しなければならないが、せいぜいfilterでやることがnullを弾く程度の簡易なものであれば十分頼れるだろう
