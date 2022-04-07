---
title: firebaseによるAsyncStorage警告をgetReactNativePersistenceで解決する
emoji: 🦵
type: tech
topics:
  - firebase
  - reactnative
  - react
  - expo
published: true
---

expoからFirestoreを利用していると、下記のような警告が出ていて困っていた

```
Warning: Async Storage has been extracted from react-native core
and will be removed in a future release. It can now be installed
and imported from '@react-native-community/async-storage' 
instead of 'react-native'. See https://github.com/react-native-community/react-native-async-storage
```

`firebase@9.4.1`以上であれば抑止する方法が入っているようだった

## やり方

`getReactNativePersistence`というのが追加されているので、これを利用する。
* https://firebase.google.com/docs/reference/js/auth.md#getreactnativepersistence

```js
import { getApp, getApps, initializeApp } from "firebase/app"
import { getAuth, initializeAuth } from "firebase/auth"
import AsyncStorage from "@react-native-async-storage/async-storage"
import { getReactNativePersistence } from 'firebase/auth/react-native'

const initalizeFirebase = () => {
  const app = initializeApp(FIREBASE_CONFIG)
  
  // ↓この設定を追加
  initializeAuth(app, {
    persistence: getReactNativePersistence(AsyncStorage)
  })
}

```

もしかしてユーザーがログアウトされてしまうのでは？というようなことも考えたが、少なくとも試した限りは特にログアウトされるというような現象は確認されなかった


### jestがコケる

しかし残念ながら、この設定を入れると、jestが下記のようなエラーを出してしまう

`SyntaxError: Unexpected token export`

mockでカバーする方法もあったが、`jest-node-exports-resolver`を入れてしまう方が楽に解決出来そうだった

```
$ yarn add -D jest-node-exports-resolver
```

configは下記のように追加すと解決するようだった

```js
// jest.config.js
module.exports = {
  //...
  resolver: 'jest-node-exports-resolver',
  // ...
}
```

## 参考
* https://github.com/firebase/firebase-js-sdk/issues/1847#issuecomment-1041548028
* https://github.com/firebase/firebase-admin-node/issues/1465#issuecomment-949053266