---
title: zenn cliの記事作成機能っぽいのををinquirerで対話式に自作する
emoji: '👐'
type: idea
topics:
  - zenn
  - javascript
published: true
---

元々自分のブログ用に[記事生成（frontmatter生成用）のスクリプト](https://github.com/terrierscript/terrier.dev/blob/master/bin/create.js)を書いていたのでそれをzenn用にカスタマイズした。

@[tweet](https://twitter.com/terrierscript/status/1307510569423917059)

## Sourceとか

```js
$ yarn add inquirer dashify gray-matter emoji-datasource-apple
```

```js
// pacjage.json
  "scripts": {
    "new": "node bin/article.js"
  }
}
```

```js
// bin/new.js
const inquirer = require("inquirer")
const dashify = require("dashify")
const grayMatter = require("gray-matter")
const fs = require("fs")
const exec = require("child_process").exec


const EMOJI_REPLACE = "____EMOJI_REPLACE____"
const randomEmoji = () => {
  const emoji = emojiSource[Math.floor(Math.random() * emojiSource.length + 1)]
  return String.fromCodePoint(...emoji.unified.split("-").map(i => parseInt(i, 16)))
}

const convert = ({ title, slug, topics, type,date }) => {
  const dir = `drafts` // articleに直接置かずに、いったんdraftsみたいなディレクトリに置いてる
  const filename = `${dir}/${date}-${slug}.md`
  const emoji = randomEmoji()
  
  const matter = grayMatter.stringify("", {
    title,
    emoji: EMOJI_REPLACE,
    type,
    topics: topics
      .replace(/ /g, ",")
      .split(",")
      .map(i => i.toLowerCase())
      .filter(i => !!i),
    published: false
  }, {lang:"yaml"})
    .replace(EMOJI_REPLACE, emoji) // js-yamlがうまく絵文字を処理してくれないので、あとからreplaceする
  return {
    filename,
    matter
  }
}


const main = () => {
  const defaultTitle = process.argv[2]
  inquirer
    .prompt([
      {
        name: "title",
        default: defaultTitle,
        validate: (item) => {
          return item.length < 60
        }
      },
      {
        // slugはtitleからdashifyを利用して生成
        name: "slug",
        default: ({ title }) =>
          dashify(title.replace(/_/g, "-"), { condense: true }).slice(0, 40),
        validate: (item) => {
          return item.length < 40
        }
      },
      {
        name: "type",
        type: "list",
        choices: ["tech", "idea"],
        default: "tech"
      },
      {
        name: "topics",
        message: "topics(comma separated)"
      },
      {
        name: "date",
        transformer: (item) => item.replace(/\//g, "-"),
        default: new Date().toLocaleDateString("sv-SE", { timeZone: "Asia/Tokyo" }),
        validate: (v) => v.split("-").length === 3 && v.split("-").every(v => Number.isInteger(Number(v)))
      },
      {
        type: "confirm",
        name: "confirm",
        default: "Y",
        message: answer => {
          const { filename, matter } = convert(answer)
          return [
            "",
            `Filename: ${filename}`,
            "Matter:",
            `${matter}`,
            "OK?"
          ].join("\n")
        }
      }
    ])
    .then(answer => {
      if (!answer.confirm) {
        console.log("cancel")
        return
      }

      const { filename, matter } = convert(answer)
      fs.writeFileSync(filename, matter)
      exec(`code ${filename}`) // お好みでVSCodeを開く
    })
}

main()
```

slugや日付の形式にしている部分は下記の記事が参考になる
https://zenn.dev/monaqa/articles/2020-09-17-vim-zenn-command