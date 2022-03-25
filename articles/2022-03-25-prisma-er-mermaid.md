---
title: prismaのER図をmermaid形式で吐くには
emoji: 🧜
type: tech
topics:
  - prisma
  - mermaid
published: true
---

prismaはサードパーティーとしてER図を出力する[prisma-erd-generator](https://github.com/keonik/prisma-erd-generator)があるが、Githubがmermaid表示に対応した今、もはや画像で吐き出す利点がなかったので、mermaidの形式で吐きたい

## どうするか？

`prisma-erd-generator`で十分可能だった。
outputのファイルを`.md`にすればよい。

```
generator erd {
  provider = "prisma-erd-generator"
  output = "scheme.md"
}
```

こうすることで`prisma/scheme.md`などに出力される

ドキュメントにはあまり深く記載されてないが、ソースを見るとたしかにそのように処理されるようだった。
* https://github.com/keonik/prisma-erd-generator/blob/eb90bdfc4f1042ee1dee300464355bc553eac9fa/src/generate.ts#L350-L355

## その他

[Community Generators](https://www.prisma.io/docs/concepts/components/prisma-schema/generators#community-generators)に列挙されているが、テキストで管理するという点で言えば[DBMLのgenerator](https://github.com/notiz-dev/prisma-dbml-generator)も選択肢としてありえそうだ