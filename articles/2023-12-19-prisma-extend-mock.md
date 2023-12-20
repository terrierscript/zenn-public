---
title: Prisma.$extendでprismaをmock化する
emoji: 🍔
type: tech
topics:
  - prisma
  - typescript
published: true
---

以前[`$use`を利用してprismaをmock化する](https://zenn.dev/terrierscript/articles/2022-08-04-prisma-middleware-test-mock)記事を書いたが、`$use`がdeprecatedになってしまった。

代替として利用できる[`$extend`](https://www.prisma.io/docs/orm/prisma-client/client-extensions)に書き換えた場合をやってみる


# `$use`を使っていたもの

まずおさらいとして、`$use`を使っていた場合は下記のようなものだった
```ts
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

# `$extends`を使う

`$extens`を使う場合、下記のような形になる。
`$use`の場合はグローバルな`PrismaClient`に影響を与えていたが、`$extends`の場合は拡張したインスタンスのみが拡張される

```ts
const mockedPrismaClientWithExtend = () => {
  return new PrismaClient().$extends({
    query: {
      task: {
        async findUnique({ }) {
          return {
            id: "a",
            name: "hoge"
          } as Task
        }
      }
    }
  })
  return mockPrismaClient
}

test("Task test", async () => {
  const prismaClient = mockedPrismaClientWithExtend()
  const task = await getTaskById(prismaClient, "foo")
  expect(task?.name).toBe("hoge")
})
```

とりあえず全部null返したいみたいな場合は[`$allOperations`](https://www.prisma.io/docs/orm/prisma-client/client-extensions/query#modify-all-prisma-client-operations)を使うと良い

```ts
const mockPrismaClient: PrismaClient = new PrismaClient().$extends({
  query: {
    $allOperations: async () => {
      return null
    },
  }
})
```

# 型

この`$extends`、型としては`PrismaClient`の互換性がなくなってしまう。
今回はテスト内のモック目的で、特にメソッドが増えたりもしないので、`@ts-ignore`で回避するので十分そうと判断した。

```ts
// @ts-ignore
const mockPrismaClient: PrismaClient = new PrismaClient().$extends({
  ...
})
```

個人的にアプリケーションのコード中に`$extends`を使う予定はないのであまり深堀りしていないが、公式ドキュメントには[`typeof`を利用した方法](https://www.prisma.io/docs/orm/prisma-client/client-extensions#type-of-an-extended-client)が記載されている