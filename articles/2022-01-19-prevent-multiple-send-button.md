---
title: useRefとuseStateでの多重送信を防止するボタン
emoji: 🧽
type: tech
topics:
  - react
  - javascript
  - typescript
published: true
---

Reactでボタンの多重送信防止を防ぐことを考えるとちょっとハマったのでメモ


## Stateで処理する（ダメそうな例）

まず普通にstateでやってみる例を考える。

```tsx
// Stateのみの場合
export const PreventMultiSubmitButton = () => {
  const [sending, setSending] = useState(false)
  const send = async () => {
    if (sending) { 
      return
    }
    setSending(true)
    await sendData()
    setSending(false)
  }
  return <Button
    onClick={() => {
      send()
    }}
    isLoading={sending}
  >
    Send
  </Button>
}
```

しかしstateの場合、更新はReactのライフサイクルに依存するため、これは適切に二重送信が防止されない。

例えばButtonのイベントを下記のようにすると二重に送信されることが見て取れるだろう

```tsx
  return <Button
    onClick={() => {
      send()
      setTimeout(() => { 
        send()
      }, 0)
    }}
```

## Refを利用する（不完全なパターン）

次にRefを利用することを考える。

```tsx
export const PreventMultiSubmitButton = () => {
  const sendingRef = useRef(false)
  const send = async () => {
    if (sendingRef.current) {
      return
    }
    sendingRef.current = true
    await sendData()
    sendingRef.current = false
  }
  return <Button
    onClick={() => {
      send()
    }}
    isLoading={sendingRef.current}
  >
    Send
  </Button>
}
```

Refの場合、stateと違いあくまで生なデータなので、`sendingRef.current`で判定することで多重送信は防止される。

ただし、refはレンダリングに影響しない。
上記のように`<Button isLoading={sendingRef.current}>`としていてもボタンコンポーネントを適切にローディング状態にすることができない

## Refとstateを用意して同期させる例

ここまでの例を経て、stateとrefを組み合わせればひとまずは問題ない状態ができそうだ。

下記にようになるだろう

```tsx
export const PreventMultiSubmitButton = () => {
  const [sendingState, setSendingState] = useState(false)
  const sendingRef = useRef(false)
  const setSending = (value: boolean) => {
    sendingRef.current = value
    setSendingState(value)
  }
  const send = async () => {
    if (sendingRef.current) {
      return
    }
    setSending(true)
    await sendData()
    setSending(false)
  }
  return <Button
    onClick={() => {
      send()
    }}
    isLoading={sendingState}
  >
    Send
  </Button>
}
```

setSendingでstateとrefを同時に更新する。
望んだ通りの結果にはなるが、多重に状態の気持ち悪さがあり、管理したい状態がもう少し増えると嫌な匂いがしそうでもう少しなんとかしてみたい

## forceUpdateを利用する

ここで重複となっているstate部分を解消するため、公式でも紹介されている[forceUpdate](https://reactjs.org/docs/hooks-faq.html#is-there-something-like-forceupdate)のテクニックを取り入れてみると割といい感じになる

```tsx
export const PreventMultiSubmitButton = () => {
  const [_ignored, forceUpdate] = useReducer(c => c + 1, 0)
  const sendingRef = useRef(false)
  const setSending = (value: boolean) => {
    sendingRef.current = value
    forceUpdate()
  }
  const send = async () => {
    if (sendingRef.current) {
      return
    }
    setSending(true)
    await sendData()
    setSending(false)
  }
  return <Button
    onClick={() => {
      send()
    }}
    isLoading={sendingRef.current}
  >
    Send
  </Button>
}
```

`const [_ignored, forceUpdate] = useReducer(c => c + 1, 0)`の部分がforceUpdateだ。stateに意味を持たせず、あくまでも再レンダリングのためだけのフラグとして扱うことで、管理する状態は減り、情報も一元化できるので個人的には好みな感じになった。

ちなみにこのテクニックは[zustand](https://github.com/pmndrs/zustand/blob/0ba3c240078cf61792a4b13ff11d774ecca70a0a/src/react.ts#L80)などでも見かけることができる