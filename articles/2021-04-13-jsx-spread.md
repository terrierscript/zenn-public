---
title: 'JSXはspread演算子で<Foo a={a} b={b}>を<Foo {...{a, b}}>で書ける'
emoji: 🍡
type: tech
topics:
  - jsx
  - react
  - javascript
  - typescript
published: true
---

# 前説

通常、JSXでパラメータを渡すならこうなる

```jsx
<Foo a={a} b={b} />
```

ただ、これらはパラメータが増えたり、それを引き回すことになるとそこそこ面倒になる

```jsx
<Foo a={a} b={b} c={c} d={d} e={e} />
```

## spread演算子を使う

まずこれらは一つの値にまとめることでspread演算子として渡す事が出来る。親から子にスルーするならばこんな具合で良い
* https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Operators/Spread_syntax

```jsx
const Baz = (props) => {
  return <Foo {...props}>
}
```

一度展開して他にも使うならこんな具合か

```jsx
const Baz = (props) => {
  const {a, b, c} = props
  return <>
    <div>{a}</div>
    <Foo {...props} />
    <Bar b={b} />
  </>
}
```

またはこんな具合だろう

```jsx
const Baz = ({a, b, c}) => {
  const props = {a, b, c} 
  return <>
    <div>{a}</div>
    <Foo {...props} />
    <Bar b={b} />
  </>
}
```

ただ若干無駄な戻しをしていたり、無駄な値を渡してしまうケースもあるだろう

# 本題

## spread演算子とShorthand property namesを組み合わせる。

上記で

```js
const props = {a, b, c} 
```
と記載しているが、これはShorthand property namesと呼ばれるものになる。

* https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Operators/Object_initializer#new_notations_in_ecmascript_2015

これをspreadと組み合わせて`{...{a, b, c}}`と書くことで下記のように記載できる

```jsx
const Baz = ({a, b, c}) => {
  return <>
    <div>{a}</div>
    <Foo {...{a, b, c}} />
    <Bar b={b} />
  </>
}
```

### うれしさ1: brace省略が出来る

この書き方だとbrace省略が出来る。うれしい

```jsx
const Baz = ({a, b, c}) => (<>
    <div>{a}</div>
    <Foo {...{a, b, c}} />
    <Bar b={b} />
  </>
)
```

### うれしさ2: 無駄なプロパティを渡さなくて良い

`props = { a, b, c　}`のようにまとめていると、なんとなく必要のないパラメータまで渡したりして不便な事があった。この書き方だとうまくその問題を除去出来る

```jsx
const Baz = ({a, b, c}) => (<>
    <NeedAandB {...{a, b}} />
    <NeedBandC {...{b, c}} />
  </>
)
```
### うれしさ3: 名前変更も楽にできる

一部propertyの名前を変える必要があっても対応しやすい

```jsx
const Baz = ({a, b, c}) => (<>
    <NeedAandB {...{apple: a, banana: b, c}} />
    <NeedBandC {...{b, c}} />
  </>
)
```
