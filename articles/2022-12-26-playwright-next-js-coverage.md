---
title: playwrightでnext.jsのcoverageまで取りたい
emoji: 🦛
type: tech
topics:
  - nextjs
  - playwright
  - nyc
  - istanbul
  - typescript
published: true
published_at: 2022-12-26 08:00
---

playwrightでnext.jsのテストを実行してcoverageまで取るのがわりと情報揃ってなかったのでまとめる。

# 準備

とりあえず普通にnext.js側のファイルを用意

```
$ yarn add next
```

```tsx
// pages/index.ts

import Head from 'next/head'
import React, { useState } from 'react'

export default function Home() {
  return (
    <Box>
      <Head>
        <title>Index Page</title>
        <link rel="icon" href="/favicon.ico" />
      </Head>
      <Box>Hello</Box>
      <ToggleButton />
    </Box>
  )
}
```

次にplaywrightの準備。istanbulを利用してくれる`playwright-test-coverage`も利用する

```
$ yarn add @playwright/test playwright-test-coverage
```

テストはまず簡素に訪問するものを作成。

```ts
// import { expect, test } from "@playwright/test"
import { expect, test } from "playwright-test-coverage"

test("index", async ({ page }) => {
  await page.goto("/")　// URLはplaywright.config.jsで指定する予定なので、`/`のみとする（後述）
  await expect(page).toHaveTitle("Index Page")
  await page.screenshot()
})
```

ここでカバレッジのために`@playwright/test`の代わりに`playwright-test-coverage`を利用する

# nyc周りの設定・next.config等の設定

カバレッジに必要な`nyc`と、babelでコードも対応させる必要があるのでそのあたりもインストールする

```
$ yarn add nyc babel-loader babel-plugin-istanbul
```

next.jsは`babel.confg.js`を追加すると勝手にそちらを読んでくれるのだが、これを行うとSWCが無効化されてしまう。カバレッジのためだけにSWCが無効化されるのも気持ち悪いので、`next.config.js`で切り替えれるようにする。

今回はenvとして`COVERAGE=1`などがある場合にカバレッジモードで起動するようにしてみる

```js
// next.config.js
module.exports = () => {
  if (process.env.COVERAGE) {
    const coverageConfig = {
      distDir: ".next_coverage",
      webpack: (config, options) => {
        config.module.rules.push({
          test: /\.(js|jsx|ts|tsx)$/,
          loader: 'babel-loader',
          options: {
            cacheDirectory: true,
            plugins: ['istanbul'],
            presets: ["next/babel"],
          },
        })
        return config
      }
    }

    return {
      ...coverageConfig
    }
  }
  return {}
}
```

webpackの設定は[next.jsのDiscussion](https://github.com/vercel/next.js/discussions/30174#discussioncomment-2421511)を参考にした。

`distDir`は通常の起動と分けておかないとたまにおかしな挙動をしてしまうので分離しておく

今回はカバレッジの起動コマンドもscriptに定義しておくことにする。ポートは何でも良いが、今回は例として3001を利用する

```json
// package.json
  "scripts": {
    "dev": "next dev",
    "dev:coverage": "COVERAGE=1 nyc next dev -p 3001",
  }
```

ついでに`nyc.config.js`も設定しておく

```js
module.exports = {
  all: true,
  include: ["src"],
  reporter: ["html", "json", "text"]
}
```

`.gitignore`も設定する

```
.next_coverage/
.nyc_output/
```

# playwrightとnycをつなぐ

playwrightには`webServer`という設定があるので、これを利用する

```js
// playwright.config.js
import type { PlaywrightTestConfig } from '@playwright/test'

const config: PlaywrightTestConfig = {
  webServer: {
    command: 'yarn dev:coverage',
    url: 'http://localhost:3001/',
    reuseExistingServer: false,
  },
  use: {
    baseURL: 'http://localhost:3001/',
  },

}

export default config
```

こうするとplaywrightを実行時に勝手に`dev:coverage`を実行してそちらを利用してくれる。
もしカバレッジを取らないplaywright実行もしたい場合はこちらもenv等を受け取って切り替えるようなことをすると良いだろう

最後に実行コマンドを追加

```json
// package.json
  "scripts": {
    // ...
    "e2e:coverage": "playwright test; nyc report"
  }
```

`nyc report`を実行しないとカバレッジが作成されないので、最後にこれを実行する

あとはカバレッジファイルを開けば閲覧出来るはず

```
$ open coverage/index.html
```
