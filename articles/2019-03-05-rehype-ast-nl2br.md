---
title: gatsby-remark-componentとrehypeでnl2br的な事をする
emoji: 🎗
type: tech
topics:
  - gatsby
  - rehype
  - remark
  - javascript
published: true
terrier_raw_title: Gatsbyでgatsby-remark-componentを使いつつrehypeのASTをいじくってnl2br的な事をする
terrier_date: '2019-03-05T12:05:37.693Z'
terrier_export_from: 'https://blog.terrier.dev/blog/2019/20190305210537-rehype-ast-nl2br'
---

> 記事作成日: 2019/03/05

GatsbyでMarkdownを記述すると、普通に改行しても反映されない。Markdownの仕様としてはそれで正しいのだが、気を使いたくないブログだとちょっと面倒さがある。

そんな時に[gatsby-remark-component](https://www.gatsbyjs.org/packages/gatsby-remark-component/)を見つけた。


まず一般的なGatsbyのMarkdown利用の場合、`gatsby-transformer-remark`を通して生のHTMLとして記事が取得出来る。そのHTMLを下記のように`dangerouslySetInnerHTML`で表示する事になる。

```js
<div dangerouslySetInnerHTML={{ __html: post.html }} />
```

`dangerouslySetInnerHTML`を使うのはまあまあ気持ち悪いのもあるし、拡張性も高くない。
そこで`gatsby-remark-component`の出番になる。

使い方としては他のプラグインと同じでconfigに追記する

```js
plugins: [
  {
    resolve: "gatsby-transformer-remark",
    options: {
      plugins: ["gatsby-remark-component"]
    }
  }
]
```

ここからちょっと特殊だ。
このpluginはあくまでrehypeのASTを`htmlAst`として返してくれるだけので、そこからは自分で変換する必要がある。
また取得のためにGraphQL側も変更が必要だ

```jsx
import rehypeReact from "rehype-react"

// 変換用の関数。
const renderAst = new rehypeReact({
  createElement: React.createElement,
}).Compiler

const BlogPost = ({ data }) => {
  const { markdownRemark: post } = data
  const content = renderHtmlAST(post.htmlAst)
  // return <Blog><div dangerouslySetInnerHTML={{ __html: post.html }} /></Blog>
  return <Blog>
    {content}
  </Blog>
}

export const pageQuery = graphql`
  query BlogPostByID($id: String!) {
    markdownRemark(id: { eq: $id }) {
      // ...
      id
      html
      htmlAst // GraphQLも追加が必要
      // ...
    }
  }
`
```


## nl2brの変換をする

ASTが返ってくればあとは`\n`を見つけて変換してやればいい。
今回は`unist-util-visit`を利用する形にした。

```jsx
import highlight from "rehype-highlight"
import unified from "unified"

const nl2br = () => {
  const transformer = (tree) => {
    visit(tree, (node, index, parent) => {
      if (node.type !== "text") return node
      if (parent.tagName !== "p") return node
      const values = node.value.trim().split("\n")
      if (values.length < 1) {
        return
      }
      const children = values
        .map((v, i) => {
          return i == 0
            ? [{ type: "text", value: v }]
            : [{ type: "element", tagName: "br" }, { type: "text", value: v }]
        })
        .flat()
      // .reduce((a, b) => [...a, ...b], []) // TODO: Array.prototype.flat

      const newChildren = [
        ...parent.children.slice(0, index),
        ...children,
        ...parent.children.slice(index + 1)
      ]
      parent.children = newChildren
    })
    return tree
  }
  return transformer
}

const renderAst = new rehypeReact({
  createElement: React.createElement
}).Compiler

export const renderHtmlAST = htmlAst => {
  // せっかくなのでunifiedに習ったやり方にした
  const tree = unified()
    .use(highlight)
    .use(nl2br)
    .runSync(htmlAst)
  // console.log(tree)
  return renderAst(tree)
}

```

Arrayのspreadでの合成は[Redux](https://redux.js.org/recipes/structuring-reducers/immutable-update-patterns#inserting-and-removing-items-in-arrays)の公式ドキュメントを参考にした。
