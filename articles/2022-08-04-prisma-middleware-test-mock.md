---
title: prismaのmiddlewareでモックを返してテストする
emoji: 🌥
type: tech
topics:
  - prisma
  - typescript
published: true
---

:::message alert
`$use`については`deprecated`となったので、この記事はすでに古い情報です。
代替として利用できる[`$extends`を利用したバージョン](https://zenn.dev/terrierscript/articles/2023-12-19-prisma-extend-mock)を記載したので、そちらを参考にしてください
:::

Prismaの絡む部分でテストしたい時に[Middleware](https://www.prisma.io/docs/concepts/components/prisma-client/middleware)の機能を使えばテスト用DBやjestのmockなど使えばできることがわかった。

## 例

テスト対象になる関数は下記のように`PrismaClient`を引数に取るようにしておくと都合が良い

```ts
// task.ts
import { PrismaClient } from '@prisma/client'

export const getTaskById = (prisma: PrismaClient, id: string) => {
  return prisma.task.findUnique({ where: { id } })
}
```

あとは`$use`にてMiddlewareを設定したprismaClientを使うようにすると良い

```ts
// task.test.ts

const mockedPrismaClient = () => {
  const prismaClient = new PrismaClient()
  prismaClient.$use(async (params, next) => {
    return {
      id: "foo",
      name: "hoge"
    } as Task
  })
  return prismaClient
}

test("Task test", async () => {
  const prismaClient = mockedPrismaClient()
  const task = await getTaskById(prismaClient, "foo")
  expect(task?.name).toBe("hoge")
})
```

例えば`Task`のテーブルはmockを返してそれ以外は普通に処理させたいという場合はこのような感じで可能

```ts
prismaClient.$use(async (params, next) => {
  if (params.model === "Task") {
    return {
      id: "a",
      name: "hoge"
    } as Task
  }
  return await next(params)
})
```

他にも`params`には`args`や`action`など一通りのパラメータが入ってきているので、条件に合わせてmockを返すなども可能そうだ