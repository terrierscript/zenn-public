---
title: playwrightのsnapshotでmask以外の方法で要素を隠す
emoji: 👺
type: tech
topics:
  - playwright
  - typescript
published: true
---


Playwrightのスクリーンショットのテストをする際、差分として考慮したくない部分に関してmaskを指定できる。

```ts
test('snapshot', async ({ page }) => {
  await page.goto('https://playwright.dev/')
  await expect(page).toHaveScreenshot({
    mask: [
      page.locator(".navbar__title.text--truncate"),
      page.locator(".getStarted_Sjon"),
    ]
  })
})
```
![](https://storage.googleapis.com/zenn-user-upload/3c8c3a77d760-20250131.png)

ある程度のランダムに発生する部分はこれで対処可能なのだが、トーストなどこれだけだと考慮しきれないケースがあり、消す方法を考えた

# パターン1: removeで行う

要素を指定して`remove()`で消すことを考えた

```ts
test('snapshot', async ({ page }) => {
  await page.goto('https://playwright.dev/')
  const target = [
    ".navbar__title.text--truncate",
    ".getStarted_Sjon"
  ]
  await Promise.all(target.map(async (selector) => {
    await page.$eval(selector, (element) => {
      element.remove()
    })
  }))
  await expect(page).toHaveScreenshot()
})
```

下記のように要素を消すことはできる。

![](https://storage.googleapis.com/zenn-user-upload/23df06cfcb35-20250131.png)


ただ見て分かる通り、他の要素の位置がずれるなどが発生してしまう。`positon:fixed`な要素などであればこれで十分対処できそうだったが、基本的にはあまり使えないケースが多いだろう

# パターン2: style属性で非表示にする

次にstyle属性で`opacity: 0`にする方法。

```ts
test('snapshot', async ({ page }) => {
  await page.goto('https://playwright.dev/')
  const target = [
    ".navbar__title.text--truncate",
    ".getStarted_Sjon"
  ]
  await Promise.all(target.map(async (selector) => {
    await page.$eval(selector, (elements) => {
      elements.setAttribute("style", "opacity: 0")
    })
  }))
  await expect(page).toHaveScreenshot()
})
```

![](https://storage.googleapis.com/zenn-user-upload/7f9db539918f-20250131.png)

これはこれでうまく行ったのだが、`$eval`を実行してからsnapshotを取るまでの間に対象の要素が出てきてしまう場合があり、これもイマイチうまくいかなかった


# パターン3: addStyleTagでCSSを追加する

上記で足りなかったので、`addStyleTag`で追加する手段を取った。

```ts
test('snapshot', async ({ page }) => {
  await page.goto('https://playwright.dev/')
  const target = [
    ".navbar__title.text--truncate",
    ".getStarted_Sjon"
  ]
  const hiddenCss = target.map(selector => `${selector} { opacity: 0 !important; }`).join("\n")
  await page.addStyleTag({ content: hiddenCss })
  await expect(page).toHaveScreenshot()
})
```

これであれば、後から要素が追加されても問題なかった。
スナップショット撮影後に戻す場合も下記のような形で行えた

```ts
test('snapshot', async ({ page }) => {
  await page.goto('https://playwright.dev/')
  const target = [
    ".navbar__title.text--truncate",
    ".getStarted_Sjon"
  ]
  const hiddenCss = target.map(selector => `${selector} { opacity: 0 !important; }`).join("\n")
  const styleHandle = await page.addStyleTag({ content: hiddenCss })
  await expect(page).toHaveScreenshot()
  // 追加した要素を消す
  await styleHandle.evaluateHandle((el) => el.childNodes.forEach(el => el.remove()))
})
```

# おまけ: blurでぼかす

発展型として、maskの代わりにblurにする事もできる。

```ts
test('snapshot', async ({ page }) => {
  await page.goto('https://playwright.dev/')
  const target = [
    ".navbar__title.text--truncate",
    ".getStarted_Sjon"
  ]
  const hiddenCss = target.map(selector => `${selector} { filter: blur(5px);  }`).join("\n")
  await page.addStyleTag({ content: hiddenCss })
  await expect(page).toHaveScreenshot()
})
```

![](https://storage.googleapis.com/zenn-user-upload/4cba4146cf5a-20250131.png)

少し分かりづらいがうっすらボケているのがわかる

「全部隠したくはないけどmaxDiffPixelsを調整してそれなりに近いと嬉しい」みたいな場合に使えるかもしれない
