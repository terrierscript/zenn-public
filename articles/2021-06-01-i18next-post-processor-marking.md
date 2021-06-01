---
title: i18nextで処理済みのテキストをUI上で少しだけ見やすくするPostProcessorを仕掛ける
emoji: 🐾
type: tech
topics:
  - i18n
  - javascript
  - i18next
published: true
---

多言語化対応時、どこを多言語化処理したか画面上からわかりやすくしたかった。

## PostProcessorを作る

`i18next`の場合、pluginの仕組みがあり、今回の要望はPostProcessorで処理できた

* https://www.i18next.com/misc/creating-own-plugins


Processor自体は下記ぐらいの簡素なもので済んだ。

```tsx
import i18n, { PostProcessorModule, TOptions } from "i18next"

const markingProcessor: PostProcessorModule = {
  type: 'postProcessor',
  name: 'marking',
  process: (value: string, _key: string, _options: TOptions, _translator: any) => {
    /* こんにちは -> [こんにちは] と表示する*/
    return `[${value}]`
  }
}
```

こうすると変換された結果が`[こんにちは]`のように表示されるので、UI上から確認して作業が進めやすくなるPluginとなる。

## PostProcessorを使う

あとは初期化時に`use`で指定し、`postProcess`として指定すれば利用できる。

```tsx
i18n
  .use(markingProcessor)
  .init({
    // ...
    postProcess: ["marking"]
    // ...
  })
```

実際は開発時のみになるはずなので、下記のように`process.env.NODE_ENV`等を見て処理するのが良いだろう

```tsx
.init({
    // ...
    postProcess: process.env.NODE_ENV === "development" ? ["marking"] : []
    // ...
  })
```

## ちょっと応用

例えばキー名も一緒に出したいならこんな具合だろう

```tsx
const markingProcessor: PostProcessorModule = {
  type: 'postProcessor',
  name: 'marking',
  process: (value: string, key: string, _options: TOptions, translator: any) => {
    // const resolved = translator.t(key)
    /* return manipulated value */
    return `[${key}:${value}]`
  }
}
```

また、キーが存在しているかどうかでマークを変えるならこんなことも出来るだろう

```tsx
const markingProcessor: PostProcessorModule = {
  type: 'postProcessor',
  name: 'marking',
  process: (value: string, key: string, _options: TOptions, translator: any) => {
    const exist = translator.exists(key) ? "✅" : "❌"
    return `${exist}:${value}`
  }
}
```

ちなみに`translator`は`typeof i18n`に近いが微妙に過不足があり、公式の型定義上現在`any`となっている。正確なところを知るなら[`Translator.js`](https://github.com/i18next/i18next/blob/2482b6117f7aa6a17ee04d24c4dfc6e46cc2138b/src/Translator.js)を見るのが一番正確になってしまっている。
