---
title: Reactで複数のchildrenを扱いたい場合の実装パターン
emoji: 🚸
type: tech
topics:
  - react
  - javascript
published: true
---

::: message
この記事は[Reactで複数のchildrenを扱いたい場合の実装パターン](https://qiita.com/terrierscript/items/3afee284621bbe126332)(2018/10)から移植したものです
:::

# 導入：Reactの`children`
Reactのchildrenを使うことで、子要素をwrapするような子要素を記述することが出来る。railsのviewで言えばyieldみたいなもの。

```jsx
const SomeLabel = ({children}) => {
  return <div>{children}</div>
}

// 呼び出し側
<SomeLabel>
  // 子要素の拡張に対して、`SomeLabel`は関心を持たなくて良い。
  <SomeDecoration>Hello</SomeDecoration>
<SomeLabel>
```

これを覚えるとだいぶReactをきれいに書きやすくなるイメージがある。

# 本題：複数の子要素を柔軟に扱いたいときにどう考えるか？

例えば何かtitle, author,bodyを貰って記事を表示するような`<Post>`を考える。

```jsx
<Post title={"aaa"} author={"bob"} body={"ほげほげ"} />
```

ここでは一番単位の大きそうな`body`をメインの`children`とすること考えてみる。

```jsx
<Post title={"aaa"} author={"bob"} >
  <div>ほげほげ</div>
</Post>
```

このままだと、bodyの部分は加工がしやすくなったが、`title`を加工するならどうするか？ということを考える。

## 手法1. そのままにpropsでElementを定義する
一番ライトな手法。

呼び出し側は若干辛いが、そのままでも加工は出来る。

```jsx
const MyPost = () => (
  <Post title={<BazDecorate>Titleですよ</BazDecorate>} author="bob">
    <FooDecorate>ほげ</FooDecorate>
  </Post>
)
```

```jsx
const Post = ({title, author, children}) => {
  return <div>
    <div>{title} : {author}</div>
    <div>{children}</div>
  </div>
}
```

titleの部分を切り出す事で多少見栄えはマシになる。
パフォーマンスは悪そう。

```jsx
const MyPost = () => {
  const titleNode = (<BazDecorate>Titleですよ</BazDecorate>)
  return <Post title={titleNode} author="bob">
    <FooDecorate>ほげ</FooDecorate>
  </Post>
}
```


### 手法2. childrenの配列を展開する
要素が管理しきれる場合であれば、childrenの配列を展開するやり方もある。

```js
const MyPost = (
  <Post>
    <BazDecorate>Titleですよ</BazDecorate>
    <BazDecorate>bob</BazDecorate>
    <FooDecorate>ほげ</FooDecorate>
  </Post>
)
```

コンポーネント側はこんな感じ

```jsx
const Post = ({children}) => {
  return <div>
    <div>{children[0]}</div>
    <div>{children[1]}</div>
    <div>{children[2]}</div>
  </div>
}
// 多分propTypes書いとかないと破綻しがち
Post.propTypes = {
  children: React.PropTypes.arrayOf(React.PropTypes.node),
}
```

Spread記法を利用するともう少しきれいに書ける。

```jsx
// Objectの中のarrayを展開するのでちょっと記法が奇抜かも。
const Post = ({ children: [ title, author, body ] }) => {
  return <div>
    <div>{title}</div>
    <div>{author}</div>
    <div>{body}</div>
  </div>
}
```

保守性の面であまり良く無いかもしれないが、そこそこには使える可能性がある。

### 手法2-2. keyで抽出

配列順序ではなく`key`を利用して組み立てる。
react-addons-css-transition-groupなどは`key`を利用してアニメーションをさせているところから思いついた。

そこそこにハックしている方法なので、多分あまりやらないほうが良いと思われる。

```jsx
const Post = ({children}) => {
  const title = children.find( (item) => item.key === "title")
  const body = children.find( (item) => item.key === "body")
  return <div>
    <div>{title}</div>
    <div>{body}</div>
  </div>
}
const MyPost = () => (
  <Post>
    <div key="title">title</div>
    <div key="body">body</div>
  </Post>
)
```

## 手法3. render関数、またはcomponent指定として外に持たせる

手法2だけではカバーできない複雑さの場合には、関数やComponentそのものを渡す、という解法がある。
かなり複雑になってくるが、汎用性だけは高い。

例えばtitleは`author`と`title`を受け取ってTitle部分として描画したいという場合を考えてみると、この方法が有益になってくる。

`title`のレンダリングを`titleRender`というpropsにする。

```jsx
const MyPost = () => (
  <Post author="bob"
    title="hoge"
    titleRender={({author, title}) => <div>{title} ({author}さんが書きました)</div>} >
    <FooDecorate>ほげ</FooDecorate>
  </Post>
)
```

```jsx
const Post = ({title, author, children, titleRender}) => {
  return <div>
    // titleRenderを関数として呼び出し返す。
    <div>{ titleRender({title, author) }</div>
    <div>{ children }</div>
  </div>
}

// titleRenderのデフォルトを設定しておきたければdefaultPorps
Post.defaultProps = {
  titleRender : ({title, author}) => (<div>{title} {author}</div>)
}
```

titleRenderに渡す部分は、Stateless Functional Componentになっているので、別途Component定義してしまっても良い。

```jsx
const SomeTitle = ({title, author}) => {
  return <div>{title} (author: {author})</div>
}
const MyPost = () => (
  <Post author="myname"
    title="hoge"
    titleRender={SomeTitle} >
    <FooDecorate>ほげ</FooDecorate>
  </Post>
)
```

Class Componentを渡さなければいけない、という場合なら`titleComponent`としてComponentを渡すのが良い

```jsx
// jsxの仕組み上、Componentで記載するためにPascalCaseにする
const Post = ({title, author, children, titleComponent: TitleComponent}) => {
  return <div>
    <div>
      // コンポーネントをjsxとして再記載。
      <TitleComponent {...{title, author}} />
    </div>
    <div>{children}</div>
  </div>
}

// 利用側
class SomeTitleClassCmp extends Component{
  render(){
    const {title, author} = this.props
    return <div>{title} (author: {author})</div>
  }
}

const MyPost = () => (
  <Post author="myname"
    title="hoge"
    titleComponent={SomeTitleClassCmp} >
    <FooDecorate>ほげ</FooDecorate>
  </Post>
)
```

この方法は[React Router@v4](https://github.com/ReactTraining/react-router/tree/v4/)のソースなどからヒントを得た。
React Routerでは`<Match>`コンポーネントなどで利用されている。

ライブラリを作る場合でもなければここまでの手法は不要かもしれない

### 欄外：ChildrenをStateless Functionで扱う

今回の本筋とはずれるが、[React Router@v4](https://github.com/ReactTraining/react-router/tree/v4/)を見ていて見つけて面白かったやり方。

ここまで`titleRender`などをStateless Functionとしていたが、実は`children`をまるごとStateless Functionにも出来る。

```jsx
const Post = ({children, ...rest}) => {
  if (typeof children == 'function') {
    return children(rest)
  }
  return <div>
    {children}
  </div>
}

const MyPost = () => (
  <Post author="bob" title="foo" body="hoge">{ ({author, title, body}) => {
    return <div>
      <div>{title}</div>
      <div>{author}</div>
      <div>{body}</div>
    </div>
  } }
  </Post>
)
```

この例だけでは「どこに使い所があるんだろう？」となってしまうが、React Routerでは、[`<Link>`をカスタムする場合](https://react-router.now.sh/custom-link-component)に有益になる。
(参考：React-router: [`<Link>`](https://github.com/ReactTraining/react-router/blob/v4/modules/Link.js) )

## 手法4. 割り切ってパターンを限定素して別なコンポーネントにする

上記の`titleRender`や`titleComponent`あたりはかなり汎用性を持たせるものになっていたが、ライブラリを作っているわけでもない限りそこまでは不要な事が多く、2、3種類パターンが作れれば良いのが現実としては多い。

### 手法4-1. exportしないprivateなコンポーネントを定義

愚直に基底となるコンポーネントを作り、`export`するのは加工をしたコンポーネントとの組み合わせたものだけにする。

```jsx
// 基底となるコンポーネントはexportしない。
const PrivatePost = ({title, author, children, titleRender}) => {
  return <div>
    <div>{ titleRender({title, author}) }</div>
    <div>{ children }</div>
  </div>
}


// 何らかの加工パターン
const FooTitle = ({title, author}) => (<div>Title: {title} Author: {author}</div>)

const BazTitle = ({title, author}) => (<div> {title} ( {author} )</div>)

// 加工済みコンポーネントをexportとする

export const FooPost = (props) => {
  return <PrivatePost {...props}
    titleRender={FooTitle} />
}

export const BazPost = (props) => {
  return <PrivatePost {...props}
    titleRender={BazTitle } />
}
```

わかりやすくはあるが、何かの拍子で`PrivatePost`をexportされたりすると管理不全になりそうな部分は注意しないといけない。

### 手法4-2. Higher Order Componentにする

Higher Order Componentで書くとだいぶすっきり書ける。

```jsx
// titleRenderの部分を受けてComponent化するHigher Order Component
const postItem = (titleRender) => {
  return ({title, author, children}) => {
    return <div>
      <div>{ titleRender({title, author}) }</div>
      <div>{ children }</div>
    </div>
  }
}

const FooTitle = ({title, author}) => (<div>Title: {title} Author: {author}</div>)
export const FooPost = postItem(FooTitle)

const BazTitle = ({title, author}) => (<div> {title} ( {author} )</div>)
export const BazPost = postItem(BazTitle)
```

手法4-1の場合、`PrivatePost`を隠蔽されるように気にする必要があったが、この方法であれば`postItem`を`export`しても問題はない。
逆に必要があればexportしてしまっても問題ない。`postItem`を単体テストするのも容易になっている。

bodyとtitleどちらも加工するようなパターンにも拡張しやすい。

```jsx
const postItem = (titleRender, bodyRender ) => {
  return ({title, author, body}) => {
    return <div>
      <div>{ titleRender({title, author}) }</div>
      <div>{ bodyRender({body}) }</div>
    </div>
  }
}

const FooTitle = ({title, author}) => (<div>Title: {title} Author: {author}</div>)
const FooBody = ({body}) => (<div>{body}</div>)
export const FooPost = postItem(FooTitle, FooBody)
```

# まとめ
* 個人的なおすすめとしては手法4のHigher Order Components。
* 手法1でも十分やれる事は多いが、可読性やパフォーマンスに難あり。
* 更に大変なことになるなら手法3を考えてみる。
* 手法2は多分そんなにやらない気がする。
