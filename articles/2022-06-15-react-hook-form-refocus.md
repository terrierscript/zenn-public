---
title: react-hook-formで投稿したら再度フォーカスするフォームを作る
emoji: 🐣
type: tech
topics:
  - react
  - typescript
  - reacthookform
published: false
---

チャット的なUIでreact-hook-formを利用する際、submitしたらもう一度inputフォームにフォーカスするようなのをやりたくなった。

![capture](https://user-images.githubusercontent.com/13282103/173836508-dfe85cd9-9e23-4c2a-afee-217f52bdc492.gif)

## 出来上がったコード

```tsx
export const RefocusInput: FC<{
  onSubmit: (value: string) => void
}> = ({ onSubmit }) => {
  const { register, handleSubmit, formState, setFocus, resetField } = useForm<{ value: string }>()
  
  useEffect(() => {
    setFocus("value")
  }, [formState.submitCount])
  
  return <form onSubmit={handleSubmit(async (data) => {
    await onSubmit(data.value)
    resetField("value")
  })}>
    <div>
      <input
        disabled={formState.isSubmitting}
        {...register("value")} />
      <button
        disabled={formState.isSubmitting}
        type="submit"
        aria-label={"submit"} >
        submit
      </button>
    </div>
  </form>
}
```

## やってること

まず送信部分で`resetField`することで入力を消している。本当は成功を確認してからのほうが良いかもしれない

```tsx
<form onSubmit={handleSubmit(async (data) => {
  await onSubmit(data.value)
  resetField("value")
})}>
```

フォーカスする部分は`formState.submitCount`で送信回数が変わる事を起点にしている。

```tsx
useEffect(() => {
  setFocus("value")
}, [formState.submitCount])
```
`resetField`ではなく`reset({value:""})`を利用した場合、`formState.submitCount`は0->1->0と変更され、利用できないようだった。


別解として、`useWatch`で値が消えたタイミングでフォーカスするという手法もあった。

```tsx
useEffect(() => {
  if (!watchValue) {
    setFocus("value")
  }
}, [watchValue])
```

こちらはちょっと泥臭いのであまり使うケースは無さそうだ