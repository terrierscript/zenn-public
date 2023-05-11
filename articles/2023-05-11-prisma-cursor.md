---
title: prismaでcursorと同等のことを自前する
emoji: 👉
type: tech
topics:
  - prisma
  - typescript
  - sql
published: true
---

Prismaはpaginationの機能として`skip`を利用するOffset paginationと`cusor`を利用するCursor-based paginationがある。

* https://www.prisma.io/docs/concepts/components/prisma-client/pagination

今回のような検証をするために、PrismaClientはlog出力設定をしておく

```ts
const prisma = new PrismaClient({
  log: ['query']
})
```
Cursor-basedを利用する場合、下記のように指定する。

```ts
const cursorId = 10
const tasks = await prisma.task.findMany({
  orderBy: {
    id: "desc"
  },
  cursor: {
    id: cursorId
  }
})
```
[ドキュメントにある通り](https://www.prisma.io/docs/concepts/components/prisma-client/pagination#-cons-of-cursor-based-pagination)、基本的に、idはソート可能な事が前提として作られている

この場合、下記のようなクエリが生成される

```sql
SELECT
  `main`.`Task`.`id`,
  `main`.`Task`.`name`,
  `main`.`Task`.`createdAt`
FROM
  `main`.`Task`
WHERE
  `main`.`Task`.`id` <= (
    SELECT
      `main`.`Task`.`id`
    FROM
      `main`.`Task`
    WHERE
      (`main`.`Task`.`id`) = (?)
  )
ORDER BY
  `main`.`Task`.`id` DESC
LIMIT ? OFFSET ?
```

WHEREの中でサブクエリとして発行している。
IDがソート可能な前提とはいえ、なかなか好みの分かれるクエリが生成される。

また、下記のように`createdAt`などcursorに利用される以外の値を利用したい場合には更に問題を起こすクエリが発行される。

```ts
const cursorId = "04c44c77-6491-4343-a832-c242647d1d51"

const tasks = await prisma.task.findMany({
  orderBy: {
    createdAt: "desc"
  },
  cursor: {
    id: cursorId
  }
})
```

```sql
SELECT
  `main`.`Task`.`id`,
  `main`.`Task`.`name`,
  `main`.`Task`.`createdAt`
FROM
  `main`.`Task`
WHERE
  `main`.`Task`.`createdAt` <= (
    SELECT
      `main`.`Task`.`createdAt`
    FROM
      `main`.`Task`
    WHERE
      (`main`.`Task`.`id`) = (?)
  )
ORDER BY
  `main`.`Task`.`createdAt` DESC
LIMIT
  ? OFFSET ?
```

## Cursorを自前で作成する

やっていることはカーソルのデータを取得して、それを利用するように分解すれば良いので、クエリを分解する

```ts
const cursorId = "04c44c77-6491-4343-a832-c242647d1d51"
const cursorTask = await prisma.task.findUniqueOrThrow({
  where: {
    id: cursorId
  }
})

const tasks = await prisma.task.findMany({
  where: {
    createdAt: {
      lte: cursorTask.createdAt
    }
  },
  orderBy: {
    id: "desc"
  },
})
```