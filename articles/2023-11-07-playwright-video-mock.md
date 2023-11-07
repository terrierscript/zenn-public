---
title: playwrightのmock動画をテストごとに切り替えたい
emoji: 🎥
type: tech
topics:
  - playwright
  - typescript
published: true
---

playwrigthtでWebカメラを利用する部分を検証したい場合、`args`として`use-file-for-fake-video-capture`を利用することができる。

* https://github.com/microsoft/playwright/issues/4532
* https://qiita.com/github0013@github/items/7969691b2d22b95bc115

この機能については上記のような記事が先行で存在している。

ただこれらは[`launghOptions`](https://playwright.dev/docs/api/class-testoptions#test-options-launch-options
)であるため、`playwright.config.js`に設定するのが基本となる。

しかしこのままだと一つの動画ファイルしか利用できなくなってしまう。

## test.useを利用して解決する

configの設定について、[`test.use`](https://playwright.dev/docs/api/class-test#test-use)で上書きする方法がある。

```ts
test.use({
  launchOptions: {
    args: [
      "--use-fake-ui-for-media-stream",
      "--use-fake-device-for-media-stream",
      `--use-file-for-fake-video-capture=vide.mjpeg`,
    ],
  },
})
```

共通化するなら下記のようなやり方になるだろう

```ts
// test-lib.ts
export const getFakeVideoOptions = (filePath) => {
  return {
    launchOptions: {
      args: [
        "--use-fake-ui-for-media-stream",
        "--use-fake-device-for-media-stream",
        `--use-file-for-fake-video-capture=${filePath}`,
      ],
    },
  }
}
```

呼び出しは下記のようにファイルごとに行う必要がある

```ts
// home1.test.ts
test.use({
  ...getFakeVideoOptions("video-mock1.mjpeg")
})
```

```ts
// home2.test.ts
test.use({
  baseUrl: "http://example.com",
  ...getFakeVideoOptions("video-mock1.mjpeg")
})
```
