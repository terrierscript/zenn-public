---
title: Prisma2でDB Resetを無理やり行う
emoji: 🧨
type: tech
topics:
  - prisma
  - prisma2
  - javascript
published: true
---

Prisma2において、開発中に`prisma db push --force --preview-feature`を行うとmigrationを無視してDBを同期することができる。
これは開発中には便利だ。

しかし例えばすでにデータが入っていてrequiredなどの整合性が取れない変更の場合エラーが出てしまうことがある

```graphql
datasource db {
  provider = "postgres"
  url      = env("DATABASE_URL")
}
generator client {
  provider = "prisma-client-js"
}

model User {
  id    Int     @id @default(autoincrement())
  email String  @unique
  name  String?
  // ↓必須の項目を追加
  someRequiredValue  String
}
```

```
$ prisma db push --force --preview-feature
Error:
⚠️ We found changes that cannot be executed:

  • Added the required column `someRequiredValue` to the `User` table without a default value. There are 1 rows in this table, it is not possible to execute this migration.

```

そして残念ながら現在DB Resetの機能はPrismaには無い。

## 仮想的なreset用スキーマで回避する

そこでリセット用のスキーマを別途用意することで回避できることを発見した。

例えば`prisma/reset.prisma`のように名前をつけて、下記のようなファイルを作成する

```graphql
// prisma/reset.prisma
datasource db {
  provider = "postgres"
  url      = env("DATABASE_URL")
}
generator client {
  provider = "prisma-client-js"
}

model Reset {
  id        Int     @id @default(autoincrement())
}
```

これを`--schema`パラメータを付与して実行する

```
$ prisma db push --preview-feature --force --schema=./prisma/reset.prisma
```

これでDBは他のテーブルが削除され、`Reset`テーブルだけが残る状態になる。
あとは再度`$ prisma db push --force --preview-feature`を行えば良い。

必要があれば下記のように`package.json`に付与しても良いだろう

```json
"scripts": {
  "db:reset": "prisma db push --preview-feature --force --schema=./prisma/reset.prisma; prisma db push --force --preview-feature"
}
```
