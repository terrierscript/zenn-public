---
title: JavaScriptでObject.entries/fromEntriesでreduceの利用頻度を減らす
emoji: 🐞
type: tech
topics:
  - javascript
published: true
---

## entriesとは

JavaScriptには`Object.entries()` / `Array.prototype.entries()` など、イテレーションができるものには`entities`というものが実装されている。


これは要素を`[key, value]` の配列として変換するものだ。

また、`Object.fromEntries`というものがある。これは逆に`[key,value]`で構成された配列をObjectへ変換するものになる。

Array / Map / Setはprototype実装されているので、`someArray.entries()`などできるが、Objectはそういうものを持てないので、`Object.fromEntries(obj)`とする必要がある

詳しくはMDNを参照してほしい

* https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Object/entries
* https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/entries
* https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Map/entries
* https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Set/entries

## Object.fromEntriesを使ってreduceの利用回数を減らそう

例えば `[{id: "foo", title: "hoge"}, {id: "baz", title: "fuga"}]`を`{ foo: "hoge", baz: "fuga"}`みたいにやりたいことは少なくない。そういう場合

```js
const target =[{id: "foo", title: "hoge"}, {id: "baz", title: "fuga"}]

target.reduce((acc, cur) => {
  return {...acc, [cur.id]: cur.title}
} , {})
```

のように`reduce`を利用することがよくある。ただreduce本来の目的ではないので、なかなかややこしくなってて可読性が辛くなる。

こういうときに`Object.fromEntries`が使える

```js
const target =[{id: "foo", title: "hoge"}, {id: "baz", title: "fuga"}]
Object.fromEntries(target.map((obj) => [obj.id, obj.title]))
```
## MapをObjectに変換する

~~同じようなことをmapでやりたいとき、もう少し工夫が必要になる~~


```js
const mapItem = new Map()
mapItem.set("foo", "hoge")
mapItem.set("baz", "fuga")
Object.fromEntries(mapItem.entries())
```

`Map`型はややこしいことに`.map`関数を持っていないので、~~代わりに`.entries()`を利用することになる~~

**追記**
[コメント](https://zenn.dev/terrierscript/articles/2021-04-02-java-script-object-entities#comment-f5723b1cae6534)より。
この例は`Object.fromEntries(mapItem)`でOKだった。

```js
const mapItem = new Map()
mapItem.set("foo", "hoge")
mapItem.set("baz", "fuga")
Object.fromEntries(mapItem)
```

### ちなみにMapをmapしたいときは

`mapItem.entries()`では`Iterator`が返ってくるので、`mapItem.entries().map`も出来ない。

そこで`Array.from()`で配列に変換する

```js
const mapItem = new Map()
mapItem.set("baz", {title: "hoge"})
mapItem.set("foo", {title: "hoge"})

Object.fromEntries(
  [...mapItem.entries()].map(( [key,value]) => [key,value.title])
  // spreadをしている部分は、下記と同義
  // Array.from(mapItem).map
)
```

## entriesでMapに変換する

ObjectをMapに変換したいときもentriesは使える。

```js
const obj = { foo: "hoge", baz: "fuga"} 

const mapItem = new Map(Object.entries(obj))
//　const mapItem = new Map(obj) はエラーになる
```

## 配列をuniqueにしたいとき

よくあるreduceの利用として行われがちだった重複を取り除きたいときにも使えるだろう。

```js
const arr = ["dog","cat","dog","dog","elephant"]

Object.keys(arr.reduce((acc, cur) => {
  return {...acc, [cur]: true}
}, {}))
```


これを`fromEntries`を使うなら

```js
const arr = ["dog","cat","dog","dog","elephant"]

Object.keys(Object.fromEntries(arr.map( (item) => [item, true])))
```

とはいえこの場合だと`Set`で

```js
const arr = ["dog","cat","dog","dog","elephant"]

const unique = [...new Set(arr)]
```
と記載できるので、こっちの方が良いだろう。

* https://github.com/you-dont-need/You-Dont-Need-Lodash-Underscore#_uniq


## ObjectなのかMapなのかSetかわからないものを変換するなら

ObjectだったりMapだったりSetだったりが混ざったりするケース（あまりない）の場合はこんな具合になる。
（当然Object -> Objectの処理は割と無駄なことをやっている）

```js
const objectOrMapToObject = (item) => {
  const itemEntries = typeof item.entries === "function"
    ? item.entries()
    : Object.entries(item)
  return Object.fromEntries(itemEntries)
}
```

`Map`だけが混ざるとわかっているなら、[`item instanceof Map`](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Operators/instanceof)などの判定も使えるはずだ。[^1]
例外的に`entries`の関数を持つようなobjectがあるとコケる。もう少しちゃんとやりたければ[lodashのisMap](https://lodash.com/docs/4.17.15#isMap)などを利用するのが良いだろう


[^1]: https://twitter.com/ymmooot/status/1377909770888257538