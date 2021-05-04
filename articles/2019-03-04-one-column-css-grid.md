---
title: シングルカラムレイアウトをCSS Gridで書くとmarginとさよなら出来て良い
emoji: 👆
type: tech
topics:
  - css
  - grid
published: true
terrier_date: '2019-03-04T14:35:22.583Z'
terrier_export_from: 'https://blog.terrier.dev/blog/2019/20190304233522-one-column-css-grid'
---

> 記事作成日: 2019/03/04



CSSで中央寄せの1カラムレイアウトをしたい場合、例えばmarginを使って下記のような事をしていたと思う

```css
.one-column-layout {
  margin: 0 auto;
  width: 700px
}
```
なんとなく言わんとしてる事はわかるのだが、「なんでここでmarginなんだ」ということはわりとモヤっとしたことは誰しもあるだろう。

CSS Gridを使うとこの点がもっと直感的に記述出来る

```css
.one-column-layout {
  display: grid;
  grid-template-areas: 
    ". header  ."
    ". body    ."
    ". footer  .";
  grid-template-columns: 1fr 3fr 1fr;
}

```

templateの分で少々記述量は増えてしまうのはあるが、`grid-template-columns`はそれぞれのカラム幅を定義していて非常に直感的だ。
また`fr`という単位もそれぞれの相対値で計算してくれるというのが良い。([fr単位について詳しくはこちら](https://developer.mozilla.org/ja/docs/Web/CSS/CSS_Grid_Layout/Basic_Concepts_of_Grid_Layout#The_fr_Unit))

ただ、いまいちここは把握しきれていなのだが、使っていてたまに内容物がはみ出すケースがあった。
そういうときはminmaxを利用することで解決出来る。`min-width`と`max-width`がかかっているのに近い挙動をしてくれる

```css
.one-column-layout {
  display: grid;
  grid-template-areas: 
    ". header  ."
    ". body    ."
    ". footer  .";
  grid-template-columns: 1fr minmax(700px, 3fr) 1fr;
}
```

