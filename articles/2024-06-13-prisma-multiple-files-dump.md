---
title: prismaのmultiple filesの変更差分をdumpしてうまいこと検出したい
emoji: 🪬
type: tech
topics:
  - prisma
  - sql
  - typescript
published: true
---

Prismaのv5.15より[schemaを複数ファイルに分割して管理できるようになった](https://www.prisma.io/blog/organize-your-prisma-schema-with-multi-file-support)。

```schema
generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["prismaSchemaFolder"]
}
```

とても便利なのだが、うっかり変な変更をしたりしてしまってないかなど確認する仕組みを作りたくなったので考えてみた。

今回はPlanetScaleの利用など、通常`db push`する環境を想定している

# primsa migrate diffでダンプする

schmaからCreate Tableを吐き出すのには、`prisma migrate diff`が利用できる

```sh
$ npx run prisma migrate diff --from-empty --to-schema-datamodel ./prisma/single/schema.prisma --script
```

これで下記のようなSQLが出力される８

```sql
-- CreateTable
CREATE TABLE "User" (
    "id" SERIAL NOT NULL,
    "name" TEXT NOT NULL,

    CONSTRAINT "User_pkey" PRIMARY KEY ("id")
);

-- CreateTable
CREATE TABLE "Post" (
    "id" SERIAL NOT NULL,
    "title" TEXT NOT NULL,
    "body" TEXT NOT NULL,
    "userId" INTEGER NOT NULL,

    CONSTRAINT "Post_pkey" PRIMARY KEY ("id")
);

-- CreateTable
CREATE TABLE "Comment" (
    "id" SERIAL NOT NULL,
    "postId" INTEGER NOT NULL,
    "body" TEXT NOT NULL,
    "userId" INTEGER NOT NULL,

    CONSTRAINT "Comment_pkey" PRIMARY KEY ("id")
);

-- AddForeignKey
ALTER TABLE "Post" ADD CONSTRAINT "Post_userId_fkey" FOREIGN KEY ("userId") REFERENCES "User"("id") ON DELETE RESTRICT ON UPDATE CASCADE;

-- AddForeignKey
ALTER TABLE "Comment" ADD CONSTRAINT "Comment_postId_fkey" FOREIGN KEY ("postId") REFERENCES "Post"("id") ON DELETE RESTRICT ON UPDATE CASCADE;

-- AddForeignKey
ALTER TABLE "Comment" ADD CONSTRAINT "Comment_userId_fkey" FOREIGN KEY ("userId") REFERENCES "User"("id") ON DELETE RESTRICT ON UPDATE CASCADE;
```

あとはこれを保存しておけば良いのだが、ファイルを分割した場合、テーブルが記述順になってしまい、不必要なdiffが発生してしまう。
（または、`1-foo.schema`, `2-baz.schema`のように順序が確定できるという手法もある）

# ダンプ結果の順序を修正する
ひとまず順序を揃えてしまうことを目的にしたスクリプトを書くことにした。

下記パッケージを利用する
* [node-sql-parser](https://www.npmjs.com/package/node-sql-parser)
  * SQLのパース
* [sql-formatter](https://www.npmjs.com/package/sql-formatter)
  * SQLのフォーマット


```ts
// bin/format.ts
import { format } from 'sql-formatter'
import SqlParser from 'node-sql-parser'

// パイプラインから取得する
const getStdin = async () => {
  const buffers = []
  for await (const chunk of process.stdin) {
    buffers.push(chunk)
  }
  return Buffer.concat(buffers).toString()
}

const main = async () => {
  const database = 'postgresql' // 利用するデータベースに合わせて変える
  const sql = await getStdin()
  const parser = new SqlParser.Parser()
  const ast = await parser.astify(sql, {
    database
  })

  if (!Array.isArray(ast)) throw new Error('ast is not an array')

  // toSorted は node v20以上のみなので注意
  const sortByTable = ast.toSorted((astA, astB) => {
    // ちょっと手抜きだが、順序さえ合えば良いので、JSON.stringifyを基準にする
    if (astA.type === astB.type) {
      return JSON.stringify(astA) > JSON.stringify(astB) ? 1 : -1
    }
    // 特に必須ではないが、気持ち的にcreateを先にしてalterをあとにしたいので並べ替え
    if (astA.type === 'create') return -1
    if (astB.type === 'create') return 1
    return 0
  })
  const sortedCreateTable = parser.sqlify(sortByTable)
  
  // formatは必須ではないが、差分がある場合に見やすさのために行う
  const formatted = format(sortedCreateTable)
  console.log(formatted)
}

main() 
```

これを使って下記のように実行する

```sh
$ npx prisma migrate diff --from-empty --to-schema-datamodel ./prisma/single/schema.prisma --script | npx tsx bin/format.ts 
```

あくまで実行用ではなく差分確認用であることに注意

```sql
CREATE TABLE `Comment` (
  "id" SERIAL NOT NULL,
  "postId" INTEGER NOT NULL,
  "body" TEXT NOT NULL,
  "userId" INTEGER NOT NULL,
  CONSTRAINT `Comment_pkey` PRIMARY KEY ("id")
);

CREATE TABLE `Post` (
  "id" SERIAL NOT NULL,
  "title" TEXT NOT NULL,
  "body" TEXT NOT NULL,
  "userId" INTEGER NOT NULL,
  CONSTRAINT `Post_pkey` PRIMARY KEY ("id")
);

CREATE TABLE `User` (
  "id" SERIAL NOT NULL,
  "name" TEXT NOT NULL,
  CONSTRAINT `User_pkey` PRIMARY KEY ("id")
);

ALTER TABLE `Comment` ADD CONSTRAINT `Comment_postId_fkey` FOREIGN KEY ("postId") REFERENCES `Post` ("id") ON DELETE RESTRICT ON UPDATE CASCADE;

ALTER TABLE `Comment` ADD CONSTRAINT `Comment_userId_fkey` FOREIGN KEY ("userId") REFERENCES `User` ("id") ON DELETE RESTRICT ON UPDATE CASCADE;

ALTER TABLE `Post` ADD CONSTRAINT `Post_userId_fkey` FOREIGN KEY ("userId") REFERENCES `User` ("id") ON DELETE RESTRICT ON UPDATE CASCADE
```

# CIに組み合わせる

あとはCIで差分があった場合に検出されるように`git diff --exit-code`を組み合わせる

```json
"scripts": {
  "test:migrate": "prisma migrate diff --from-empty --to-schema-datamodel ./prisma/schemas/ --script | tsx ./bin/format.ts > prisma/dump.sql; git diff --exit-code ./prisma/dump.sql",
}
```

