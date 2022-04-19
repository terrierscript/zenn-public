---
title: なるべく怠けるAPI周りの型付け on next.js
emoji: 🦥
type: tech
topics:
  - nextjs
  - typescript
  - react
published: true
---

管理画面や生存期間の短い試験機能など「あんまり頑張りたくないけどanyを多少減らす程度には型ほしい」みたいな時などにちょこちょこ使ってるテクニックが溜まってきたのでまとめてみる。

今回はnext.jsのAPIに絞って記述しているが、おそらく他でも使えるはず。

## レスポンスの返り値を`Awaited<ReturnType typeof someFunction>`で怠ける

APIのレスポンスが定まり切らなかったり流動的な部分で動的にする場合、`Awaited<ReturnType typeof ...>`の組み合わせが便利。

```tsx

// こんな関数があるとして
// const getFooData = async (id): FooData
// const getBazData = async (id): BazData

// 型を取れるようにレスポンスの関数を切り出し
const someComplexResponse = async (id) => {
  const foo = await getFooData(id)
  const baz = await getBazData(id)
  return { foo, baz }
}

export type SampleApiResponse = Awaited<ReturnType<typeof someComplexResponse>>
// => { foo: FooData, baz: BazData }

const handler: NextApiHandler<SampleApiResponse> = async (req, res) => {
  const response = await someComplexResponse(req.query.id)
  res.json(response)
}

export default handler

```

```tsx

export const useSomeSampleApi = () => {
  const { data, error } = useSWR<SampleApiResponse>("/api/sample?id=foo", fetcher)
  // dataにSampleApiResponseの型がつく
}

```

## POSTの受け取りデータはzodの`z.infer`でちょっとだけ怠ける

POSTなどでの受け取りはある程度ちゃんとやるしか無いが、その中でも個人的には[`zod`](https://github.com/colinhacks/zod#objects)で`z.infer`するのが一番ラクに感じた

```ts
import { NextApiHandler } from "next"
import { z } from "zod"

// スキーマを定義
const AnimalPostScheme = z.object({
  name: z.string(),
  age: z.number(),
  kinds: z.enum(["dog", "cat"])
})

//`z.infer`で型を取り出せる
export type AnimalPostRequest = z.infer<typeof AnimalPostScheme>

const handler: NextApiHandler = async (req, res) => {
  try {
    const data = AnimalPostScheme.parse(req.body)
    const animal = await createAnimal(data)
    res.json({ animal })
  } catch (e) {
    res.status(400).end()
  }
}

export default handler

```

データ作成に利用した値をそのまま取り出せるのであればこういうことになるだろう

```ts
const handler: NextApiHandler<AnimalPostRequest> = async (req, res) => {
  const animal = await getAnimal(req.query.id)
  res.json(animal)
}
```

## クエリパラメータの`string|string[]`を`[value].flat(1)`で怠ける

最後にクエリパラメータの処理。
これはいくらか乱暴なので使用箇所には注意。

next.jsのAPIのクエリパラメータは`string|string[]`と見分ける必要がある

通常は下記のように`typeof`にするのが順当だ。
不特定に公開している場合やユーザーからの値を受け入れる場合はもちろん上記のように適切に処理するべきだろう

```ts
const handler: NextApiHandler = async (req, res) => {
  const id = req.query.id
  const targets = req.query.targets
  if (typeof id !== "string") {
    res.status(400).end()
    return
  }
  if (Array.isArray(targets)) {
      res.status(400).end()
      return
  }
  ...
```

一方管理画面などアクセスを制限でき、更にDynamic Routingによって(`/api/books/[id].ts`,`/api/books/[[...idList]].ts`)`string`か`string[]`かがほぼ変わらないようなケースならもう少し手を抜きたい。

`string`にしたい場合は`toString`やtemplate litralを利用する手抜きが考えられるだろう

```ts
const handler: NextApiHandler = async (req, res) => {
  const book = await getBook(req.query.id.toString())
  // ...
```
```ts
const handler: NextApiHandler = async (req, res) => {
  const book = await getBook(`${req.query.id}`)
  // ... 
```

ただ`foo,baz`など`join`した挙動になってしまうのはちょっと不安が残る。

そこで一度配列にしてしまって取り出す

```ts
const handler: NextApiHandler = async (req, res) => {
  const [id] = [req.query.foo].flat(1)
  const book = await getBook(id)
  // ...
```
これであれば配列の先頭が来るので、多少安心感がある

`string[]`に寄せたい場合も同様配列化して`flat`する手法が使える

```ts
const handler: NextApiHandler = async (req, res) => {
  const targets = [req.query.targets].flat(1)
  const items = await getItems(targets)
  // ...
```
