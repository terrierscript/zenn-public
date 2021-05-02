---
title: redux-observableで検索機能の実装を写経してRxJSのパワーを感じる
emoji: 💪
type: tech
topics:
  - javascript
published: true
terrier_export_from: https://qiita.com/terrierscript/items/c672ddad4c24936b1061
---

::: message
この記事は[Qiitaの記事](https://qiita.com/terrierscript/items/c672ddad4c24936b1061)から移植したものです
:::

# まえがき

[TAKING ADVANTAGE OF OBSERVABLES IN ANGULAR](http://blog.thoughtram.io/angular/2016/01/06/taking-advantage-of-observables-in-angular2.html)
という記事が自分Rxへの理解を深めそうだったので、こちらをベース理解を深めたい(以下上記記事を元記事と呼ぶ）

今回は元記事をreact + Redux + redux-observable で実装してにrxjsと仲良くなることを目的とする

## redux-observableとRxJSに対する個人的なステータス

* Reduxでの非同期処理ミドルウェアとして[redux-obsevable](http://redux-observable.js.org)の良さわかってきた
    * APIが非常にシンプル。actionCreatorがすっきりする
        * 1つのaction発火時に複数のことやったり、複数種のactionに共通の処理やったりするのがすっきりする
    * 1度発火したactionは書き換えさせないという思想は結構扱いやすくて良い
* でもRxわからないマン
    * redux-observableのために非同期を解決するぐらいのためのRxは使える
    * ああ、「[Everyting is Stream](http://slides.com/robwormald/everything-is-a-stream#/)」ってジャックラッセルテリアが言ってるやつね。知ってる知ってる みたいな状態。
        * RxがどうReduxの単方向なデータ構造でどう恩恵をもたらすのかがしっくり来てない

## 元記事の要約

* [TAKING ADVANTAGE OF OBSERVABLES IN ANGULAR](http://blog.thoughtram.io/angular/2016/01/06/taking-advantage-of-observables-in-angular2.html)
* Inputに入力すると、即時に検索して表示するものを作る場合を考える
* この時、大きく以下３つの問題に当たる
    1. キーストローク毎にリクエストするのはよろしくない
    2. 同じリクエストで済むのに無意味にリクエスト飛ばすのはよろしくない
        * foo -> fooo -> foo -> fooo -> foo みたいな入力をした場合に不必要にデータ飛んでしまう可能性
    3. 複数リクエストを発火した場合の処理
        * A -> Bの順番にリクエストしたら B -> Aの順で返ってきちゃって表示がおかしくなる場合
* Anglar を扱ったサンプルの記事

# 実装
## 前準備

### api.js
元記事でWikipediaのopensearchを利用しているので、同じような感じで実装。
API層は独立させてpromise返すようにしておく。

```js
import fetchJsonp from 'fetch-jsonp'
import qs from 'querystring'

export const searchApi = (word) => {
  const baseURL = "https://ja.wikipedia.org/w/api.php"
  const params = {
    action: "opensearch",
    format: "json",
    search: word
  }
  const url = `${baseURL}?${qs.stringify(params)}`
  return fetchJsonp(url)
    .then( response => response.json() )
    .then( json => json[1])
}
```

### action / reducer
Reduxの共通部分は素直に作る。changeInputとloadResultという２つのactionがあるだけ。

```js
// action.js
export const CHANGE_INPUT = "CHANGE_INPUT"
export const LOAD_RESULT = "LOAD_RESULT"

// キー入力変更時
export const changeInput = (input) => ({
  type: CHANGE_INPUT,
  payload: input
})

// API結果取得時
export const loadResult = (result) => ({
  type: LOAD_RESULT,
  payload: result
})
```

```js
// reducer.js
import { LOAD_RESULT, CHANGE_INPUT } from './actions'
const result = (state = [], action ) => {
  switch(action.type){
  case LOAD_RESULT:
    return action.payload
  default:
    return state
  }
}

const word = (state = "", action ) => {
  switch(action.type){
  case CHANGE_INPUT:
    return action.payload
  default:
    return state
  }
}

export const reducers = { result, word }
```

### component / container

こちらも普通に表示 & イベントフックするだけ

```jsx
import React from 'react'
import { Provider } from 'react-redux'
import { connect } from 'react-redux'
import { configureStore } from './store'
import { changeInput } from './actions'

const SearchPedia = ( { word, result, changeInput }) => {
  return <div>
    <input value={ word } onChange={ e => changeInput(e.target.value)} />
    <h3>Result</h3>
    <ul>
      {result.map( (r, i) => <li key={i}>{r}</li>)}
    </ul>
  </div>
}

// Container
const SearchPediaContainer = connect(
  state => state, 
  { changeInput }
)(SearchPedia)

// Provider
export default class App extends React.Component{
  constructor(){
    super()
    this.store = configureStore()
  }
  render(){
    return (
      <Provider store={this.store} >
        <SearchPediaContainer />
      </Provider>
    )
  }
}
```

## その1. middlewareでpromiseベースな実装

まずrxjs / redux-observable使わない場合をやってみる。
observableの対比のため、今回はthunkやpromise使わず素middlewareでやっている。

```jsx
// middleware.js

import { CHANGE_INPUT, loadResult } from './actions'
import { searchApi } from './api'

// 普通はあまりmiddlewareに直接実装を書く事はしないが、今回は説明用ということでこうする
export const searchMiddleware = store => next => action => {
  if(action.type === CHANGE_INPUT){ 
    // `CHANGE_INPUT`のときだけ別途検索処理を発火させる
    searchApi(action.payload)
      .then( result => {
        store.dispatch(loadResult(result))
      })
  }
  return next(action)
}
```

```js
// store.js
import { createStore, combineReducers, applyMiddleware } from "redux"
import { reducers } from './reducers'

export const configureStore = () => {
  return createStore(
    combineReducers(reducers, {}),
    applyMiddleware(searchMiddleware)
  )
}
```

### 完成品
[DEMO](https://terrierscript.github.io/example-redux-observable-search/#/middleware)

> ![](https://storage.googleapis.com/zenn-user-upload/eeqamq9v8hz8y11pg3oajg53r09s)

入力毎に通信されているような形になっているのがわかる。元記事で上げられている3つの問題を解決するとなると、なかなか大変そうだ。

## その2. redux-observableで実装
やっとこさ本題。

redux-observableのmiddlewareセットアップをする

```js
// store.js
import { createStore, combineReducers, applyMiddleware } from "redux"
import { createEpicMiddleware } from "redux-observable"
import { reducers } from './reducers'
import { epics } from './epic'

export const configureStore = () => {
  return createStore(
    combineReducers(reducers, {}),
    applyMiddleware(createEpicMiddleware(epics))
  )
}
```

```js
// epic.js
import 'rxjs'

import { CHANGE_INPUT, loadResult } from './actions'
import { combineEpics } from 'redux-observable'
import { searchApi } from './api'

// とりあえず元となるepicを作る。(action$) => action$ と返すと無限ループ化するので
// ignoreElementsで何もしないようにする
const searchEpic = (action$) => action$.ofType(CHANGE_INPUT)
    .do(action => console.log(action)) // debug用
    .ignoreElements()

export const epics = combineEpics( searchEpic)
```
`do`と`ignoreElements`のデバッグは[このStackoverflow](http://stackoverflow.com/questions/40408041/redux-observable-epic-that-doesnt-send-any-new-actions)あたりを参考にした

ここからepicのobservableを除々に拡張させていくようにする

#### Phase1: CHANGE_INPUTが来たらダミーデータ返すepicを作る

とりあえず何かmock返す感じのやつを作る。debug用に`do`オペレータでconsole.logしている

```js
const searchEpic = (action$) => (
  action$.ofType(CHANGE_INPUT)
    .do(action => console.log(action)) // debug用
    .mapTo(loadResult([ "DUMMY",  "DUMMY",  "DUMMY"])) // mapToは入力に関係なく値を返す
    .do(action => console.log(action)) // debug用
)
```
こうすると、こんな感じでactionが流れてくる

```
Object {type: "CHANGE_INPUT", payload: "a"}
Object {type: "LOAD_RESULT", payload: Array[3]}
Object {type: "CHANGE_INPUT", payload: "ab"}
Object {type: "LOAD_RESULT", payload: Array[3]}
Object {type: "CHANGE_INPUT", payload: "abc"}
Object {type: "LOAD_RESULT", payload: Array[3]}
```

#### Phase2: mergeMap (旧flatMap)を利用してPromiseを解決

`mergeMap`でpromiseを返す事で、promiseを処理した結果を取得出来るようにする。
ここまででMiddlewareの場合と同等に通信出来るようになった。

```js
const searchEpic = (action$) => (
  action$.ofType(CHANGE_INPUT)
    .do(action => console.log(action)) // debug用
    .map( ({payload}) => payload )
    .mergeMap( (word) => searchApi(word) ) // Promiseを解決
    .map( result => loadResult(result) ) // 結果を返す
    .do(action => console.log(action)) // debug用
```

キー入力する度にリクエストが発行されている様子が見て取れる

```
Object {type: "CHANGE_INPUT", payload: "a"}
"CALL API" "a"
Object {type: "LOAD_RESULT", payload: Array[10]}
Object {type: "CHANGE_INPUT", payload: "ab"}
"CALL API" "ab"
Object {type: "LOAD_RESULT", payload: Array[10]}
Object {type: "CHANGE_INPUT", payload: "abc"}
"CALL API" "abc"
Object {type: "LOAD_RESULT", payload: Array[10]}
```

#### Phase3: `debounceTime`で連続した入力をまとめる (元記事：Taming the user input )

`a`, `ab`, `abc`と連続入力して毎度APIが走るのを`debounceTime`を利用して防ぐ。
rxjsっぽくなってきた。

```js
const searchEpic3 = (action$) => (
  action$.ofType(CHANGE_INPUT)
    .do(action => console.log(action)) // debug用
    .map( ({payload}) => payload )
    .debounceTime(400) // 400ms以内での入力を間引き
    .mergeMap( (payload) => searchApi(payload) )
    .map( result => loadResult(result) )
    .do(action => console.log(action)) // debug用
)
```

動かしてみると先ほどと違ってAPIコールが **キー入力を止めたタイミング** でだけ起きているのがわかる。

```
Object {type: "CHANGE_INPUT", payload: "a"}
Object {type: "CHANGE_INPUT", payload: "ab"}
Object {type: "CHANGE_INPUT", payload: "abc"}
"CALL API" "abc"
Object {type: "LOAD_RESULT", payload: Array[10]}
```

#### Phase4: `distinctUntilChanged`で値が変わった時だけ通信する (元記事：Don’t hit me twice)

同じ値での不必要なリクエストを制御する。

```js
const searchEpic = (action$) => (
  action$.ofType(CHANGE_INPUT)
    .do(action => console.log(action)) // debug用
    .map( ({payload}) => payload )
    .debounceTime(400)
    .distinctUntilChanged() // 値が変更された時だけ処理させる。
    .mergeMap( (payload) => searchApi(payload) )
    .map( result => loadResult(result) )
    .do(action => console.log(action)) // debug用
)
```

このようにして、「a -> ab -> abc (ここで待つ) -> ab -> abc」と入力してみると
APIが一度だけ走るのがわかる

```
Object {type: "CHANGE_INPUT", payload: "a"}
Object {type: "CHANGE_INPUT", payload: "ab"}
Object {type: "CHANGE_INPUT", payload: "abc"}
"CALL API" "abc"
Object {type: "LOAD_RESULT", payload: Array[10]}
Object {type: "CHANGE_INPUT", payload: "ab"}
Object {type: "CHANGE_INPUT", payload: "abc"}
// ここでAPI発行されない
```

ちなみに、`distinctUntilChanged`での値比較をしているので、下記のようにactionをそのまま渡していると、うまく動かない

```js
const searchEpic = (action$) => (
  action$.ofType(CHANGE_INPUT)
    .distinctUntilChanged() // currentAction === previewActionの比較がうまくいかない
)
```
今回のサンプルでは、予め`map( ({payload}) => payload)`として文字列比較にした。
どうしてもactionで比較したい場合があれば、[deep-equal](https://www.npmjs.com/package/deep-equal)などを使えば良さそうに思える

#### Phase5: `mergeMap` -> `switchMap` (元記事: Dealing with out-of-order responses)

`mergeMap`だったのを`switchMap`にすることで、「新しい値が来たタイミングで、前のsubscriptioをunsbscribeする」らしい。
この機能を使うと、２つ連続で発行したAPIリクエストの後に来た方を優先して処理を進めるようになるようだ。

```js
const searchEpic = (action$) => (
  action$.ofType(CHANGE_INPUT)
    .do(action => console.log(action)) // debug用
    .map( ({payload}) => payload )
    .debounceTime(400)
    .distinctUntilChanged() 
    .switchMap( (payload) => searchApi(payload) ) // switchMapにする
    .map( result => loadResult(result) )
    .do(action => console.log(action)) // debug用
)
```

正直このあたりはまだ理解が浅いと感じているが、[mergeMap](http://reactivex.io/rxjs/class/es6/Observable.js~Observable.html#instance-method-mergeMap) と [switchMap](http://reactivex.io/rxjs/class/es6/Observable.js~Observable.html#instance-method-switchMap) のそれぞれのマーブル記法を見てもそういうことで良いのだろう、という風に理解している。

下記も参考になるかもしれない

* [learn-rxjs](https://www.learnrxjs.io/)
    * [mergeMap](https://www.learnrxjs.io/operators/transformation/mergemap.html)
    * [switchMap](https://www.learnrxjs.io/operators/transformation/switchmap.html)
* [SwitchMap vs MergeMap in the #ngrx example](http://stackoverflow.com/questions/38496604/switchmap-vs-mergemap-in-the-ngrx-example)
* [RxJavaのflatMap, switchMap, concatMapを理解する](http://qiita.com/yshrsmz@github/items/1d206a4fc9bc2329bbed)
    * 個人的に非常にわかりやすかった比喩
    * rxjsでなくRxJavaだったり、mergeMapでなくflatMapだったりするが、置き換えて読める
* [What is the difference between flatmap and switchmap in RxJava?](http://stackoverflow.com/questions/28175702/what-is-the-difference-between-flatmap-and-switchmap-in-rxjava)

### 完成品 & 最終形

最終的にepicはこんな感じになった（途中まで書いてた`do`でのデバッグは消している）

```js
// epic.js

import 'rxjs'

import { CHANGE_INPUT, loadResult } from './actions'
import { combineEpics } from 'redux-observable'
import { searchApi } from './api'

const searchEpic = (action$) => (
  action$.ofType(CHANGE_INPUT)
    .map( ({payload}) => payload )
    .debounceTime(400)
    .distinctUntilChanged()
    .switchMap( (payload) => searchApi(payload) )
    .map( result => loadResult(result) )
)

export const epics = combineEpics(
  searchEpic,
)
```

[DEMO](https://terrierscript.github.io/example-redux-observable-search/#/observable)

> ![](https://storage.googleapis.com/zenn-user-upload/iewrupxu8ucyl3tnj33n3b5w3f59)
# まとめ
* わりとRxJSの心がわかった気がする。
    * なんとなくマーブル記法の意図するところが読めるようになってきた
* 単に非同期処理のためにredux-observableを使っているだけだとなかなか理解しづらいが、やはり色々手を動かすと理解しやすいと感じた。
    * 「これ何が嬉しいんだ？」というオペレータを無理矢理使ってみると色々覚えられそう
* ReduxでのRxの活かし方がちょっとわかった
    * 今回の例で言えば、Rxが無かった場合はreduxのstoreに色々変数持たせるか、コンポーネントのlocal stateで一時的に貯めたりする必要がありそう
    * 「Reduxのデータとしてstoreしたくない」みたいな場合にデータ処理を解決させる手段の一つと考えられる
    * Reduxの世界を基準に考えると、ある意味「Rxの内部で独立したstateがある」という大雑把な考え方をすると理解に繋がりそう
* 地味にデバッグ時の`do`と`ignoreElements`便利。これを知れたのが個人的に収穫だった。
* Angularの情報でもRxJavaの情報でもあまり齟齬無く活かせるのすごい。Rx軍団つよい。


