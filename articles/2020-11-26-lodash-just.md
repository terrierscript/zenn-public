---
title: lodashの代わりにjustを使う
emoji: 👨‍⚖️
type: tech
topics:
  - javascript
  - lodash
  - typescript
published: true
---

なるべくlodashを使わず標準の機能で済まそうとしている[^1]が、どうしても標準だと面倒で欲しくなるケースがある。

例えば1つの配列をn個に分ける[`_.chunk`](https://lodash.com/docs/4.17.15#chunk)などはちょうどよいだろう。


[You Dont Need Lodash Underscore](https://github.com/you-dont-need/You-Dont-Need-Lodash-Underscore)を参考に自前実装すると最低でもこのぐらいは必要だ。

```js
const chunk = (input, size) => 
  input.reduce((arr, item, idx) =>  idx % size === 0
    ? [...arr, [item]]
    : [...arr.slice(0, -1), [...arr.slice(-1)[0], item]]
  , []);
```

```js
chunk(['a', 'b', 'c', 'd'], 2);
// => [['a', 'b'], ['c', 'd']]
```

### justを使う

そこで探してみると[just](https://github.com/angus-c/just/)という関数群ライブラリがあるので紹介してみたい。

https://github.com/angus-c/just/

## 特徴

* パッケージは`just-***`という名前で単関数が独立している。必要なものだけ使える
* zero-dependencyな作り。
* 比較的簡素に作られているので自前実装に参考にしやすい

## 例

例えば上記の`_.chunk`の場合でいうと[just-split](https://github.com/angus-c/just#just-split)が対応する。

justはすべてのパッケージがそれぞれ分離されているので、`just-split`をインストールする

```sh
$ npm install just-split
```

あとはほとんど`_.chunk`と同じような関数が利用できる。

```js
import split from "just-split"

split([1, 2, 3, 4, 5, 6, 7, 8, 9], 3)
// [[1, 2, 3], [4, 5, 6], [7, 8, 9]];
```

また、justの場合はコードも下記のように独立しており簡素なため、「パッケージを入れるほどではないから自前に引っ張ってこよう」という場合も参考にしやすいだろう。
https://github.com/angus-c/just/blob/master/packages/array-split/index.js

lodashは内部でお互いのコードに依存していたりするので、持ってくるにはちょっと参考にしづらい
https://github.com/lodash/lodash/blob/master/chunk.js

同様ramdaなどもあるが、こちらも[splitEvery](https://ramdajs.com/docs/#splitEvery
)を見てみるとそこそこ内部で依存していて参考にはしづらい
https://github.com/ramda/ramda/blob/v0.27.0/source/splitEvery.js
