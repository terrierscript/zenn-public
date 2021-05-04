---
title: (2016)メンテナブルな React Component を目指すための小技集
emoji: 🩴
type: tech
topics:
  - react
  - javascript
published: true
---

::: message
この記事は[Qiitaの記事](https://qiita.com/terrierscript/items/372727448b58d4738018)から移植したものです。
一部調整はしてますが、基本的に古い内容なのでご注意ください
:::


個人的にやってみてうまくいった事中心の網羅的な小技集。

**書くこと**

* React.Componentの話
* 多分中級者ぐらい向け
* プロダクションコード向け
* 目指したい所・ゴール
    * よりデザイナ・フレンドリーなコンポーネント
    * よりテスタブルなコンポーネント
* （単一のプロダクトなど狭いスコープにおける）コンポーネントの再利用性

**[ほとんど]書かない事**

* Flux・ReduxみたいなState管理の話
* 全体的なプロジェクト構造の話
* ライブラリ向けのコンポーネントの話、再利用性の話
* パフォーマンスの事
    * パフォーマンスは大事な項目ではあるが、今回はあまり重点を置かない

---

# [Stateless Functional Component](https://facebook.github.io/react/docs/reusable-components.html#stateless-functions)で書けるだけ書く

最新のReactでは、Componentは関数として記載することが出来る。
Stateless Functional Componentと呼ばれる。
（以下、SFCと略す）

```jsx
const Foo = (props) => {
  return <div>props.baz</div>
}
```

体感で言うと、ほぼ8割ぐらいのコンポーネントはSFCで書けるなと感じた。
SFCにすることで、このあたりのメリット/デメリットがあるがある。

* メリット
    * ピュアFunctionなので可読性が高くなりやすい。
        * 考えることが減る
    * 本質的なビューレイヤーとロジックが絡むレイヤーを分離出来る
        * デザイナーとのやりとりで「ここいじって！」というのがすごくやりやすくなる
    * Storybookでの表示やスナップショットテストがやりやすい（スナップショットテストについては後述）
    * 将来的に高速されることが期待できる。

* デメリット（`class Component`を使わないと使えない物）
    * lifecycle系のメソッド（componentDidMountなど）
    * stateの管理
    * Context
    * refs
    * サードパーティライブラリのインスタンス管理


`class Component`を使う場合でも、下記に気を使いながら組み立てていく。

* 全て`class Component`にねじ込む必要はない。部分的にSFCにしたらどうだろうか？
* 親と子に分けて考えられないか？
    * 親がstateやlifecycleを面倒見る事に集中して、子は表示に集中させたりしたら良くならないか？
        * 結構このパターンでうまくいくことが多い
    * もっと小さくコンポーネントを切り出せないか？
* Hihger Order Componentで記述したら簡易に解決出来ないか？
    * [recompose](https://github.com/acdlite/recompose/) でいい感じにならないか？


# propsをspread展開(`{...props}`)して全ての値をそのまま渡すだけのコンポーネントを作る

[公式にも載っている](https://facebook.github.io/react/docs/jsx-spread.html)方法

`{...props}`という記法でObjectを展開して、子のpropsを流す手法。
覚えておくとちょこちょこ使える。

例えば下記のような場合に使えるテクニック

* 標準のDOMにちょっとクラスだけつけときたい
* WrapperなコンポーネントやHigher Order Componentを利用するときに中身をそのまま渡したい


Statleess Functionと組み合わせて素朴に使っていける。

```jsx
const MyInput = (props) => (<input className="my-input-class" {...props} />)

const SomeComponent = () => {
  return <MyInput onChange={(e) => {foo() }} />
}
```

restを利用して一部の値だけ加工することも可能

```jsx
const MyInput = ({ someValue, ...rest}) => {
  const cls = someValue === "foo" ? "baz" : "bar"
  return <input className={cls} {...props} />
}
```


# propsやstateは、関数内でなるべく引き回さず、ローカル変数に展開しておく

好き好みだと思うが、個人的に好きなやり方。

propsやstateをそのまま引き回すよりも、関数の序盤でSpread構文を利用してバラしてしまったほうが見通しが良く、メンテしやすいと感じる事が多い。

後々SFCにもしやすい。

```jsx
const Foo = ({some, values}) => {
  return <div>{some} Hello {values}</div>
}

class Foo extends Component{
  render(){
    const {some, values} = this.props
    const {foo, baz, bar} = this.state
    const {hoge, fuga} = this.context
    return <div>{some} Hello {values} {foo}</div>
  }
}
```

stateとpropsが同名の場合に使えなくなるという欠点は存在している。
しかしその場合は、既にちょっとコンポーネントが肥大化してるかも？ということに目を向けて切り出したほうが良いかを一回検討することにしている。

どうしても同名のstateとpropsを扱わなければならないなら、spread構文で別名にアサインする事も出来る。

```jsx
const {foo : fooProps } = this.props
const {foo : fooState } = this.state
const {foo : fooContext } = this.context
```


# 意味付け出来る単位で細かめにコンポーネントを切り分ける

Reactだけでなく、slimなどのテンプレート言語全般な知識だが、なるべく意味付け出来る単位があればコンポーネントはわけてしまう方がスッキリする。

```jsx
const SomeForm = ({name, onChangeName, mail, onChangeMail onSubmit}) => {
  return <from>
    <div>
      <label>
        Name: <input onChange={onChangeName} name={name} />
      </label>
      <label>
        Email: <input onChange={onChangeMail} name={mail} />
      </label>
    </div>
    <input type={onSubmit} value={"send"} />
  </from>
}
```

たとえばこんなコンポーネントなら、こんなわけ方を出来る

```jsx
const SomeInput = ({name, onChangeName, mail, onChangeMail}) => {
  return <div>
    {/* NameとEmailのinputで切り分けたり共通化しても良い。*/}
    <label>
      Name: <input onChange={onChangeName} name={name} />
    </label>
    <label>
      Email: <input onChange={onChangeMail} name={mail} />
    </label>
  </div>
}
const SomeSendButton = ({onSubmit}) => {
  return <input type={onSubmit} value={"send"} />
}

const SomeForm = (props) => {
  return <from>
    <SomeInput {...props} />
    <SomeSendButton {...props} />
  </from>
}
```

`{...props}`で不要なpropsまで渡しているのはちょっと手抜きになっているポイント。
子コンポーネントが薄い状態であればさほど問題にはならない部分ではあるが、気になるなら値を絞り込んで渡してあげること。


Reactの場合、変数で分離するだけでコンポーネントを分離できるので、コンポーネントの分離コストが安い。
なので「意味付けが出来る」ぐらいな単位でわけてしまって良いと思う。
可読性が上がるだけでなく、後々デザインなど変わる場合にも変更する単位というのを明確に考えやすいので、捨てやすく作り直しやすい。

ただし、要件が固まりきってない実装初期段階でやりすぎると逆にやりづらくなったりするので、ある程度固まってきたタイミングでリファクタするなどでも良いだろう。



# SFC内で条件分岐で分離して結果を返すのであれば、それぞれの状態を子のSFCとして切り分ける

例えばアイテムが存在しない場合に何かを出す、などの場合。

```jsx
// よくやるパターン
const SomeList = ({someItem}) => {
  if(someItem === undefined){
    return <div>アイテムが存在しません</div>
  }
  return <div>{someItem}</div>
}
```

こんな感じに書きがちだが、結構あっという間に肥大化して辛くなる

```jsx
// 辛くなったパターン
const SomeList = ({someItem}) => {
  if(someItem === undefined){
    return <div>
      <Mecha>
        <Sugoi>
          <Decoration>
          アイテムが存在しません
          </Decoration>
        </Sugoi>
      </Mecha>
    </div>
  }
  if(someItem === "fuga"){
    return <div>アイテムがfugaです</div>
  }
  return <div>
    <Sugoi>
      <Mendokusai>
        <Decoration>
          {someItem}
        </Decoration>
      </Mendokusai>
    </Sugoi>
  </div>
}
```

自分の場合は、極力条件分岐ごとでコンポーネントを切ってしまう。

```jsx
// 細かめにコンポーネント切るパターン

const ItemNotFound = () => <div>アイテムが存在しません</div>
const Item = ({someItem}) => <div>{someItem}</div>

const SomeList = ({someItem}) => {
  if(someItem === undefined){
    return <ItemNotFound />
  }
  return <Item someItem={someItem} />
}
```

こうして置くと、下記のような利点がある

* 条件分岐に対して、デザインするタイミングで意識することが減る
* Storybookなどで試しやすい
* 肥大化に対しても対処しやすい


# React wayに`children`を活用していく

子要素にあたるようなものがstringの場合、`props`で値を渡しがちだが、独自なpropertyを使わずに`children`を利用すると、スッキリ書けて良い事が多い。

https://facebook.github.io/react/docs/multiple-components.html#children

よくやリがちなパターンから見ていく。

```jsx
const FooButton = ({label}) => {
  return <button>{label}</button>
}
```

Labelを表示させる分には良いが、「ラベルの中身を加工したい！」みたいなあるあるなパターンでわりとすぐ死ぬ

```jsx
{/* propsでしか定義出来ない */}
<FooButton label="foo" />

{/* 色を設定したくなったりするとこんな感じになる */}
<FooButton label="foo" color="red"/>
{/* 拡張がどんどん増えて地獄になるパターン */ }
<FooButton label="foo" color="red" weight="bold" img="dog" animate="false"/>

```

`children`を利用するパターンだと、ある程度解決させられる。

```jsx
const GoodButton = ({children}) => {
  return <button>{children}</button>
}
```

要素で挟んでいた子要素が自動的に`children`に割り当てられる。「囲むことが必須」ではなく「囲むことも出来る」なので、propsとしてchildrenを指定することも出来る。

内部要素の装飾が増えたとしても、装飾に関しては責務を追い出すことがしやすい。

```jsx
{/* どっちもいける */}
<GoodButton children="foo" />
<GoodButton>foo</GoodLabel>

{/* 色々な装飾への依存は、別なコンポーネントに責務を移譲出来る */}
<GoodButton><FooDecorate>foo</FooDecorate></GoodLabel>
```

::: details PropTypesに関する古い内容

あわせて、PropTypesを仕掛けるなら`node`というのを仕掛けておくと、表示可能なものに限定することが出来る。

```js
GoodLabel.propTypes = {
  children: React.PropTypes.node,
}
```

:::

`children`の方がオススメではあるが、最初に例示した`<FooButton>`も`label`にnodeを渡すことで同様のことは再現出来る。このあたりは使い分け。

```js
<FooButton label={<div>item</div>} />
```

更に複雑にchildrenを扱う場合については下記に記載した。

* [Reactで複数のChildrenを扱う場合の方法を考える](http://qiita.com/inuscript/items/3afee284621bbe126332)


# Componentの実装に不安があるならcreate-react-appで素振りしてみる。

::: message alert
現在、個人的にはcreate-react-appよりも[next.jsの方が素振りにおいて楽](https://zenn.dev/terrierscript/books/2020-09-next-js/viewer/1-1)な可能性があります
:::


いきなりコンポーネントをゴリゴリ作り出すと結構難しい事がある。「どうもうまくいかない（or いかなそう）」という場合は、素振りをするのが良い。
しかし、ちょっと進めてしまった場合だと、既存のプロジェクトの上でそのあたりを試していくのは結構難しい場合が多い。

なのでまっさらな場所からスタートさせるのに、[create-react-app](https://github.com/facebookincubator/create-react-app)を使うのが良いだろう。
個人的に、（今のところ）ライフサイクルが長いproduction用途にcreate-react-appを使う事はちょっと懐疑的だが、プロト目的ならすごく良いボイラープレートツールだと思う。

```
$ npm install -g create-react-app

$ create-react-app my-prototype
$ cd my-prototype/
```

`src/App.js`というところがエントリになっている。ここに書いてもいいが、なんとなく頭をすっきりさせたい事多いので、中身をごっそり入れ替えて使う方が個人的には好き。

```jsx
import React, { Component } from 'react';
// src/SomePrototypeComponent.js に色々試すの書く
import SomePrototypeComponent from './SomePrototypeComponent'

class App extends Component {
  render() {
    return (
      <div className="App">
        <SomePrototypeComponent />
      </div>
    );
  }
}

export default App;
```

そしてstart

```
$ npm start
```

[storybook](https://getstorybook.io/)を導入して、その上でプロトタイプしてみても良いかもしれないが、storybookはプロトタイプ目的と言うよりはスタイルガイド目的の側面
のほうが強いので、そんな素振り用途には向いてないかもしれない。

# テスタブルにコンポーネントを管理する

コンポーネントのテストにどのぐらい割くべきなのか？みたいなことを考えるのはなかなか難しい課題だが、個人的には「さらっとでもやっとくと良い」ぐらいな認識。

そこの温度感にちょうどいいSnapshot Testingというのが最近にわかに出てきている。

幾つか利用方法があるので記載する

* [jest](http://facebook.github.io/jest/)のスナップショットモードを利用する
    *  [snapshot-testing](http://facebook.github.io/jest/docs/tutorial-react.html#snapshot-testing)の項目に詳しく記載してある
* [Storybook](getstorybook.io) + [StoryShots](https://github.com/kadirahq/storyshots)を利用する
    * スタイルガイドとして運用しつつ、テストとしても活用していける。
* [react-test-renderer](https://www.npmjs.com/package/react-test-renderer)を直接使う
    * jestやstoryshotsが内部的に利用しているのはこのreact-test-renderer。facebookが提供している。（react-test-renderとか類似名のパッケージがあるので注意）
    * 諸般の事情で他のテストツールに依存できない（したくない）ならこれを利用するのも一つの手だろう。

個人的にはStorybook + StoryShotsをよく使っている。
テストコマンドが分散してしまうのだが、表示確認が簡易になる部分が気に入っている


# PropTypesとは良い距離感で付き合う

::: message alert
このブロックは流石に古すぎるので閉じています。現状はTypeScriptを利用するのが良いでしょう
:::

:::details 過去の内容

[PropTypesはオワコン](https://speakerdeck.com/koba04/the-state-of-react-dot-js-2016?slide=31)、flowかTypescriptを使え！という話があったりもするが、どちらもなかなか手を出すには万人に勧めづらいハードルの高さがある現状。

今のところだいたいこんな気持ちで向き合っている

* そんなにガチガチに考えずに、ゆるーく使ったほうがよさそうな所は使う。
    * あると便利だが、こだわりすぎてもしょうがないかなという気持ち。
    * ライブラリなどを書いてるならガチガチに書いて良いと思う。
    * 普通のプロダクション利用だと全部に書いて回るのはコスパが悪いこともある
    * もしプロダクションで「全部書きたい」というならflowとかを入れるほうが将来性を考えると有効そう。
* 書くタイミングは、なるべく要件固まってからにする。
    * 要件固まってないViewは、書いておいても書き直すとかになってしまいがち。
    * 要件が固まった後に書いたほうが嬉しい事が多い。
* onXXXなどの関数ハンドラ系については積極的に書いていく 
* 書くタイミングは、なるべく要件固まってから。固まってないものは無理に書かない。
    * どうしても要件固まってないViewは、書いておいても書き直すとかになってしまうのでそんなに費用対効果良くないなと感じている。
    * 要件が固まった後に書いたほうが嬉しい事が多い。
* 同じPropTypesを一箇所にまとめる管理方法もあるので、必要なら使う
    * [react-router@v4](https://github.com/ReactTraining/react-router/blob/v4/modules/PropTypes.js)

:::
