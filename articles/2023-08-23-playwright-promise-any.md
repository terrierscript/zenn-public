---
title: Playwrightで複数いずれかの条件待ちたいとき
emoji: 🤡
type: tech
topics:
  - playwright
  - javascript
  - typescript
published: true
---

Playwrightで外部のサービスを扱ったり、表示されるページが一定で無い場合など、「AかBいずれかの条件が整ってたら次に進めたい」というようなことがまれにある。

このような場合は`Promise.any`を使うと解消できる。

```ts
await Promise.any([
  page.waitForSelector('.some-element'),
  page.waitForSelector('#some-another-element')
  page.waitForTimeout(500)
])
```

それぞれPromiseではあるので、`Promise.any`でいずれかが達成すればOKとすれば良い。

これでまず目的は達成できた。

ただ、テストは成功するが失敗した方の`waitForXXX`についてエラーログには出てしまう

```
=========================== logs ===========================
waiting for locator('.some-element') to be visible
============================================================
```

## エラーを潰したいとき

もしエラーが気になるなら、`waitForFunction`などで独自に実装するしかない

```ts
await page.waitForFunction(async () => {
  if (document.querySelector('.some-element')) {
    return true
  }
  if (document.querySelector('#some-another-element')) {
    return true
  }
  return false
})
```

ちなみにpuppeteerであれば[AbortController](https://github.com/puppeteer/puppeteer/pull/10018)が`waitForSelector`などで利用できるらしいのだが、残念ながらPlaywrightにはまだ実装されていない。
