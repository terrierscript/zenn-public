---
title: react18でtesting-libraryのwrapperにpropsを渡せない件の回避策
emoji: 🎺
type: tech
topics:
  - testinglibrary
  - react
  - typescript
published: true
---

React18になり、[testing-library](https://testing-library.com/)の[react-hooksがdeprecatedとなった](https://github.com/testing-library/react-hooks-testing-library#a-note-about-react-18-support)

基本的にはカバーされているのだが、下記のように`wrapper`を利用した場合の挙動が変わっていた

```tsx
const Wrapper = ({children, ...props}) => {
  return <TestContainer {...props}>
    {children}
  </TestContainer>
}

test("Some test", () => {
  const { result,rerender } = renderHook(() => useCounter(),{
    wrapper: Wrapper,
    initialProps: { foo: "baz"}
  })
  rerender({
    foo: "bar"
  })
})

```

このようなテストを書いた際、以前の`@testing-library/react-hooks`では`wrapper`のコンポーネントにpropsが渡されていたが、`@testing-library/react`では`wrapper`はpropsを受け取れない形となっている（そもそも型エラーになる）

それぞれソースとしては下記部分で挙動がちがうことを確認出来る

* testing-library/react-hooks
  * https://github.com/testing-library/react-hooks-testing-library/blob/3b719d1b637105eda7c7b481c9772cfbd1d52b2d/src/helpers/createTestHarness.tsx#L31-L33
* testing-library/react
  * https://github.com/testing-library/react-testing-library/blob/27a9584629e28339b9961edefbb2134d7c570678/src/pure.js#L104-L107

Contextに依存するようなhooksをテストしたい場合もあり、これはそこそこ困ることがあった

## 対策
### その1: hooks側を対応させる

きれいな解決方法としてはhooks側をtesting-libararyでやりやすいように、値を受け取って処理するhooksとcontextと依存するhooksを分離すること

例えば下記のようなhooksがあった場合

```tsx

const useMessageCount = () => {
  const messages = useMessageContext()
  const messageCount = useMemo(() => messages.length,[messages])
  const latestMessage = useMemo(() => messages[0], [messages])
  return {
    messageCount,
    latestMessage
  }
}

```

下記のような分離をする。

```tsx

const useMessageCount = (messages) => {
  const messageCount = useMemo(() => messages.length,[messages])
  const latestMessage = useMemo(() => messages[0], [messages])
  return {
    messageCount,
    latestMessage
  }
}

const useContextMessageCount = () => {
  const messages = useMessageContext()
  return useMessageCountInternal(messages)
}
```

こうすれば`useMessageCount`はこれまで同等のテストが出来るだろう。

### その2: renderHookを自前する

hooks自体を書き換えるのがなかなか初手ではやりづらいケースもあるだろう。

幸い`testing-library/react`側の`renderHook`はそれほど複雑でもないので、あまりきれいな手段ではないがpropsを受け取れるような`renderHook`を自前するというのもある。

概ねこんな具合だ。
他のオプション周りの型などは省略してしまっているのはご了承いただきたい

```tsx

export function renderHookWithPropsWrapper<Result, Props>(renderCallback: (props: Props) => Result, options: {
  wrapper: React.JSXElementConstructor<{ children: React.ReactElement } & Props>
  initialProps: Props
}) {
  const { initialProps, wrapper: Wrapper, ...restOptions } = options
  const result = React.createRef<Result>()
  function TestComponent({ renderCallbackProps }: { renderCallbackProps: Props }) {
    const pendingResult = renderCallback(renderCallbackProps)

    React.useEffect(() => {
      // @ts-ignore
      result.current = pendingResult
    })

    return null
  }

  const { rerender: baseRerender, unmount } = render(
    <Wrapper {...initialProps} >
      <TestComponent renderCallbackProps={initialProps} />
    </Wrapper>,
    restOptions
  )

  function rerender(rerenderCallbackProps: Props) {
    return baseRerender(
      <Wrapper {...rerenderCallbackProps} >
        <TestComponent renderCallbackProps={rerenderCallbackProps} />
      </Wrapper>
    )
  }

  return { result, rerender, unmount }
}
```

一部`ref`の使い方が行儀悪くなっていたりはするが、概ねこれで動作するものにはなるようだった。
