---
title: ark-uiにtailwind-animationを組み合わせる
emoji: ⛵️
type: tech
topics:
  - typescript
  - tailwind
  - css
  - html
  - react
published: true
---

Ark UIはReact等で利用できるコンポーネントライブラリだ。
特徴としてボタンのような単純な機能ではなく、Date PickerやDialogなど、状態を保有するような少し複雑なコンポーネントを中心によく使うものが揃っている点や、styleを付与してない事が挙げられる。

styleが無いのでtailwindなどのライブラリと相性が良い。一方でアニメーション等もデフォルトでは付与されていないので、このあたりは面倒を見る必要がある。

この対処としてCSSのattributeを記載してアニメーションを付与する方法が提案されている

* https://ark-ui.com/docs/overview/animation

```css
[data-scope='tooltip'][data-part='content'][data-state='open'] {
  animation: fadeIn 300ms ease-out;
}

[data-scope='tooltip'][data-part='content'][data-state='closed'] {
  animation: fadeOut 300ms ease-in;
}
```

とはいえCSSを書く量をなるべく怠けたいので`tailwind-animation`と組み合わせる方法を考えた。

## tailwind-animationと組み合わせる

今回は[`Toast`](https://ark-ui.com/docs/components/toast)の例を利用する。
Tailwind側の機能としては`group`と`Data Attribute`を利用する。

* [group class](https://tailwindcss.com/docs/hover-focus-and-other-states#styling-based-on-parent-state)
* [Data attribute](https://tailwindcss.com/docs/hover-focus-and-other-states#data-attributes)

```tsx
const [Toaster, toast] = createToaster({
  placement: 'bottom',
  duration: 3000,
  removeDelay: 100000,
  max: 1,
  // count: 10,
  render(toast) {
    return (
      <Toast.Root className='group'>
        <div className='
        group-data-[state=open]:animate-duration-500
        group-data-[state=open]:animate-fade-up
        group-data-[state=open]:animate-normal
        '>
          <div className='
          group-data-[state=closed]:animate-duration-500
          group-data-[state=closed]:animate-reverse 
          group-data-[state=closed]:animate-fade-up
          group-data-[state=closed]:animate-ease-in
        '>
            <div className='alert alert-success'>
              <Toast.Title>{toast.title}</Toast.Title>
              <Toast.Description>{toast.description}</Toast.Description>
              <Toast.CloseTrigger>Close</Toast.CloseTrigger>
            </div>
          </div>
        </div>
      </Toast.Root>
    )
  },
})
```

`group-data-[state=open]`のような複雑な指定がちょっと組み合わせになってて難しくなっているが、これは`group`と`data-state`の組み合わせになる。

`ark-ui`は[data state](https://ark-ui.com/docs/styling/overview#styling-states)という形で、`data-`のattributeにstateを付与してくれる。
Toastの場合は`data-state=open`や`data-state=closed`が付与される。

`group-`のprefixについては`group`のclass指定された親の状態によって子要素に適用したりしなかったりを切り替えられる。
これが組み合わされて`group-data-[state=open]`のような指定になっている。

今回はわかりやすくバラバラに構成したが、`Toast.Root`にopenの場合のアニメーションを指定することも出来る。

```tsx
<Toast.Root className='
  group
  data-[state=open]:animate-duration-500
  data-[state=open]:animate-fade-up
  data-[state=open]:animate-normal
  '>
    <div className='
    group-data-[state=closed]:animate-duration-500
    group-data-[state=closed]:animate-reverse 
    group-data-[state=closed]:animate-fade-up
    group-data-[state=closed]:animate-ease-in
  '>  
```

ただし、OpenとCloseで同名のアニメーションを付与すると終了時がうまく動かないケースがあるので、下記のような記載はうまくいかなかった



```tsx
// ❌ うまく動かなかった例
<Toast.Root className='
  group
  data-[state=open]:animate-duration-500
  data-[state=open]:animate-fade-up
  data-[state=open]:animate-normal
  data-[state=closed]:animate-duration-500
  data-[state=closed]:animate-reverse 
  data-[state=closed]:animate-fade-up
  data-[state=closed]:animate-ease-in
'>  
```