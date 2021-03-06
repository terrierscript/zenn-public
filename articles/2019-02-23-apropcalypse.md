---
templateKey: blog-post
title: Apropcalypseという造語に出会ったのでまとめる
emoji: 👾
type: idea
date: 2019-02-23T10:09:59.520Z
topics:
  - react
  - javascript
published: true
terrier_export_from:
  - https://blog.terrier.dev/blog/2019/20190223000000-apropcalypse/

---

> 記事作成時期:2019/02


[Defining Component APIs in React](https://jxnblk.com/blog/defining-component-apis-in-react/)というReactのコンポーネントのプラクティスを列挙している記事を見ていたところ「[Apropcalypse](https://jxnblk.com/blog/defining-component-apis-in-react/#avoid-apropcalypse)」という単語に出会った。

はて見覚えもないしなんだろうと読み進めるとこれは「
[The how and why of flexible React components](https://speakerdeck.com/jenncreighton/the-how-and-why-of-flexible-react-components-289aa486-464a-4dea-b89a-6f92d0af6606?slide=10)」のスライドで語られている造語で、「Apocalypse + props」らしい。Apocalypseは「啓示・黙示録」など出てくるがこの場合は「この世の終わり・世の終末」あたりの意味合いの方が近そう。

下記ほとんど原文のとおりだが、一応自分なりの咀嚼を記述しておくと、
これは下記のようなものを指している。

```jsx
<Button
  icon={starIcon}
  text="hello"
/>
```

で、このボタンが色々膨れていくと

```jsx
<Button
  icon={starIcon}
  afterIcon={heartIcon}
  hoverIcon={filledStarIcon}
  text="hello"
  color="red"
  primary={true}
  ...
/>
```

みたいな感じで膨れて地獄が完成していく。結構あるあるだろう

なのでchildrenをちゃんと生かしてこうしていこうという事になる

```jsx
<Button>
  <StarIcon />
  <InnerText color="red">hello</InnerText />
  <HeartIcon />
</Button>
```

