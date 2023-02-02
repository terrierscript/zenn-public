---
title: zodでネストした一部が失敗しても許可する型を作りたい場合
emoji: 🦭
type: tech
topics:
  - zod
  - typescript
  - javascript
published: true
---

例えば下記のようなスキーマを考える

```ts
const AuthorSchema = z.object({
  name: z.string()
})
const BookSchema = z.object({
  name: z.string(),
  author: AuthorSchema
})
```

この型において、`author`の型が失敗していたら無いものとして無視して扱いたいケースがある場合について、少し工夫が必要だったのでまとめる。


## optionalをつける
まず初手として単純にオプションにしたければ`optional()`をつけることになる

```ts
const AuthorSchema = z.object({
    name: z.string()
  })
const BookSchema = z.object({
  name: z.string(),
  author: AuthorSchema.optional()
})
```

が、例えば`author`が間違っている場合失敗扱いになってしまう

```ts
BookSchema.parse({
  name: "book",
  author: {
    authorName: "bob"
  }
})
// => ZodError
```

## catchをつける

失敗時のために`.catch`を組み合わせる

```ts
const AuthorSchema = z.object({
  name: z.string()
})
const BookSchema = z.object({
  name: z.string(),
  author: AuthorSchema.optional().catch(undefined)
})

BookSchema.parse({
  name: "book",
  author: {
    authorName: "bob"
  }
})
// => {name: "book", author: undefined }
```
これでパース自体はうまくいくようになる。

が、これを`z.infer`して型を取り出した場合、`author`が不足となってしまい、少し都合が悪い

```ts
type Book = z.infer<typeof BookSchema>
const book: Book = { 
  name: "foo",
} // => 型エラー
```

## 型のために更に`optional`を追加（結論）

型の問題に対処するために、更に`optional`をつける

```ts
const AuthorSchema = z.object({
  name: z.string()
})
const BookSchema = z.object({
  name: z.string(),
  author: AuthorSchema.optional().catch(undefined).optional()
})
type Book = z.infer<typeof BookSchema>
const book: Book = {
  name: "foo",
} // 型OK
```

これでまず目的は解決できた

## もう一歩綺麗にする

とはいえ流石に記述が気持ち悪いので、ちょっと微調整する

```ts
const AuthorSchema = z.object({
  name: z.string()
})
const OptionalAuthorSchema = AuthorSchema.optional()
const BookSchema = z.object({
  name: z.string(),
  author: OptionalAuthorSchema.catch(undefined).optional()
})
```

エラーの場合には`null`にしたければ下記のようにしても良い

```ts
const OptionalAuthorSchema = AuthorSchema.nullish()
const BookSchema = z.object({
  name: z.string(),
  author: OptionalAuthorSchema.catch(null).optional()
})
```

エラーの場合はちゃんと見分けれる形にしたいのであれば下記のようなことも可能

```ts
const AuthorSchema = z.object({
  name: z.string()
}).or(z.object({
  error: z.literal(true)
}))
const OptionalAuthorSchema = AuthorSchema.catch({ error: true })
const BookSchema = z.object({
  name: z.string(),
  author: OptionalAuthorSchema.optional()
})
```

Result型っぽくすることも可能だがここまでやるなら他のアプローチを考えたほうが良いかもしれない

```ts
const AuthorSchema = z.object({
  name: z.string()
})
type Author = z.infer<typeof AuthorSchema>
const AuthorResultSchema = AuthorSchema
  .transform<{ value: Author, success: true }>(param => ({
    value: param,
    success: true
  })).or(z.object({
    success: z.literal(false)
  }))
  .catch({ success: false })
type AuthorResult = z.infer<typeof AuthorResultSchema>

```