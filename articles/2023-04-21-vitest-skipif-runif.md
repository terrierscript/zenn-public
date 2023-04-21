---
title: vitestのskipIf/runIfを再利用できるようにする
emoji: 💈
type: tech
topics:
  - vitest
  - typescript
  - javascript
published: true
---

vitestには、`runIf`/`skipIf`という機能がある

* https://vitest.dev/api/#test-skipif


```ts
test.skipIf(process.env.NODE_ENV === "development")("only dev", () => {
  expect(1 + 2).toBe(3)
})
```
このように条件を指定して、実行するしないが決定される。

# skipIf / runIfを再利用する

例えば遅いテストやDBを利用するテストなど、環境によって実行するしないを切り分けたいことは少なくない。

毎回定義するのも面倒なので、再利用できるようにしたい。

`skipIf`や`runIf`は`test`と同じものを返してくれるので、ただそれを定義すればよさそうだった。

```ts
const slowTest = test.skipIf(process.env.SKIP_SLOW_TEST || process.env.CI)

slowTest("slow test", () => {
  expect(1 + 2).toBe(3)
})
```

```ts
// DBのテストの実行を切り替えたいとき
const databaseTest = test.runIf(process.env.DATABASE_URL)

databaseTest("slow test", () => {
  expect(1 + 2).toBe(3)
})

```

ちなみに`describe`でも同様に利用可能

```ts
const describeSlow = describe.skipIf(process.env.SKIP_SLOW_TEST || process.env.CI)

describeSlow("xx", () => {
  test("yy", () => {
    expect(1 + 2).toBe(3)
  })
})
```