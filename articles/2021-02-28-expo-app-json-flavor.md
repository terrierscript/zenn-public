---
title: expoのapp.jsonでflavorみたいなことをやる
emoji: 🏙️
type: tech
topics:
  - expo
  - javascript
  - reactnative
published: true
---

expoのconfigは`app.json`と並列して`app.config.js`を並べることで、動的な`app.json`を構築できる。

* https://docs.expo.io/versions/latest/config/app/
* https://docs.expo.io/workflow/configuration/

```js
// app.config.js

export default ({ config }) => { // app.jsonのexpo部分が取れる
  console.log(config.name)
  return config
}
```

ちなみに`app.json`を削除した場合`app.config.js`を読んでくれるので、こちらにすべての設定を記述することも可能だが、[react-native-version](https://github.com/stovmascript/react-native-version)などエコシステムが使えなくなったりしがちなので、`app.json`を残しつつ利用するのが良いだろう

## flavorっぽいことをしてみる

これを利用してビルドflavorみたいなことをしてみよう。

元の`app.json`がこのような状態と想定する

```json
{
  "expo": {
    "name": "example-flavor-config",
    "slug": "example-flavor-config",
    "version": "1.0.0",
    "orientation": "portrait",
    "icon": "./assets/icon.png",
    "splash": {
      "image": "./assets/splash.png",
      "resizeMode": "contain",
      "backgroundColor": "#ffffff"
    },
    "updates": {
      "fallbackToCacheTimeout": 0
    },
    "assetBundlePatterns": [
      "**/*"
    ],
    "ios": {
      "supportsTablet": true
    },
    "android": {
      "adaptiveIcon": {
        "foregroundImage": "./assets/adaptive-icon.png",
        "backgroundColor": "#FFFFFF"
      }
    },
    "web": {
      "favicon": "./assets/favicon.png"
    }
  }
}
```

そこに上書きした差分だけを記述した`app.staging.json`をこんな感じで準備してみる

```json
{
  "expo": {
    "name": "(staging)example-flavor-config",
    "slug": "example-flavor-config-staging",
    "splash": {
      "backgroundColor": "#000000"
    }
  }
}
```

JavaScriptでのobjectのマージは面倒なので、今回は[deepmerge](https://www.npmjs.com/package/deepmerge)を利用する

```
$ yarn add -D deepmerge
```

これであとは`json`を結合すれば良い

```js
// app.config.js
import merge from "deepmerge"
import stagingConfig from "./app.staging.json"

export default ({ config }) => {
  if (process.env.BUILD_FLAVOR === "staging") {
    return merge(config, stagingConfig.expo)
  }
  return config
}
```

あとはこんな感じで環境変数を差し替えることで使えるようになる

```
$ BUILD_FLAVOR=staging expo build:ios
```