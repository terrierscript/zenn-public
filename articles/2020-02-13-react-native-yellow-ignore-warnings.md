---
title: ReactNativeでignoredYellowBoxではなくYellowBox.ignoreWarningsを使う
emoji: 🟨
slug: 2020-02-13-react-native-console-ignored-yellow-box-yellow-box-ignore-warnings
type: tech
topics:
  - react
  - reactnative
  - expo
  - javascript
published: true
terrier_date: '2020-02-13T13:40:36.647Z'
terrier_export_from: https://blog.terrier.dev/blog/2020/20200213224036-react-native-console-ignored-yellow-box-yellow-box-ignore-warnings/
terrier_raw_title: React Nativeのconsole.ignoredYellowBoxではなくYellowBox.ignoreWarningsを使う

---

> 記事作成日: 2020/02/13

React NativeでfirebaseのSDKを使った場合など、`Setting a timer for a long period of time,...`のようなsetTimeoutに関するwarningが発生する。


なかなか外部ライブラリに依存していたりすると解決は難しい一方、毎回黄色いボックスでタブが埋められるのもなかなかに困る


上記エラーでは、同時に下記のissueへの案内が掲示される。
https://github.com/facebook/react-native/issues/12981

ここで下記のようなエラーをサスペンドするような方法が掲示されている。

https://github.com/facebook/react-native/issues/12981#issuecomment-302059217

```js
console.ignoredYellowBox = ['Setting a timer'];
```

しかしこの`console.ignoredYellowBox`は最新のreact-nativeでは動かなくなっている。

この情報を追いかけると下記のissueを発見した

https://github.com/facebook/react-native/issues/17503#issuecomment-357252144

ということで

```js
import {YellowBox} from 'react-native'
YellowBox.ignoreWarnings([
  'Setting a timer'
])
```

とすればひとまず隠す事ができる

公式ドキュメントでも確認することが出来た
https://facebook.github.io/react-native/docs/debugging#warnings
