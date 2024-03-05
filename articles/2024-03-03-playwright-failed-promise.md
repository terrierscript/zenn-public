---
title: playwrightで複数要素を同時にwaitして失敗パターンを早めに検知する
emoji: ⏳
type: tech
topics:
  - playwright
  - typescript
  - javascript
published: true
---

例えばtoastのような要素が成功時に出現する場合、playwrightは下記のような書き方になるだろう

```ts
const successLocator = page.locator('.success-toast')
await successLocator.waitFor()
expect(successLocator).text("成功しました")
```

ただこれだと失敗してしまった場合でも成功時の要素をタイムアウトで待つことになってしまう。
そこで失敗時に出てくる要素がある場合は、これも同時に待機すると早めに失敗を検知できる。

```ts
const failedLocator = page.locator('.failed-toast')
const successLocator = page.locator('.success-toast')
await successLocator.or(failedLocator).waitFor()
expect(successLocator).toBeVisible()
expect(successLocator).text("成功しました")
```

`successLocator.or(failedLocator)`の部分で二つの要素を待機し、`toBeVisible`で成功の要素が表示されていることで確認する。
失敗時の要素が表示されている場合はそこでコケるはずだ。
