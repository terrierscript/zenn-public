---
title: SWRのmiddlewareを使ってmockせずにテストする
emoji: 🗿
type: tech
topics:
  - jest
  - swr
  - typescript
  - javascript
published: true
---

非同期処理のテストを行う場合、モックを噛ませないとうまく行かないケースがどうしても存在する。
SWR1.0で導入されたmiddlewareを利用するとこれをいい感じに回避できることを見つけたのでまとめる

## 下準備

今回はまず下記のようなコンポーネントをテストすることを考える

```tsx
export const useDictionary = (word: string) => {
  return useSWR(`https://api.dictionaryapi.dev/api/v2/entries/en/${word}`, (url) => fetch(url).then(r => r.json()))
}

export const SearchResult: FC<{ word: string }> = ({ word }) => {
  const { data, error } = useDictionary(word)
  if (error) {
    return <div>エラーが発生しました</div>
  }
  if (!data) {
    return <div>読込中です</div>
  }
  return <div>
    <pre>
      <code>{JSON.stringify(data, null, 2)}</code>
    </pre>
  </div>
}

```

(注：今回のコンポーネントの場合、単に分離すればmockせずにテスト可能なレベルなものだが、今回は説明の簡素化のためにこうする)

## 普通のテストの場合

例えば下記のように書いてもローディング状態が取れてしまうだけで、適切にテストができない

```tsx
import renderer from 'react-test-renderer'

test("snapshot test ", () => {
  const tree = renderer.create(
    <SearchResult word="dog" />
  ).toJSON()
  expect(tree).toMatchSnapshot()
})

// 結果：
// <div>
//   読込中です
// </div>
// `;
```

このような場合は多いのは`jest`等のmockReturnVlalueを利用するなどがある。

```tsx
import useSWR from "swr"

jest.mock("swr")
// @ts-ignore
useSWR.mockReturnValue(({ data: { mock: "data" } }))

test("snapshot test ", () => {
  const tree = renderer.create(
    <SearchResult word="dog" />
  ).toJSON()
  expect(tree).toMatchSnapshot()
})

// 結果：
// <div>
//   <pre>
//     <code>
//       {
//   "mock": "data"
// }
//     </code>
//   </pre>
// </div>
// `;

```

ただやはりmockは記法が独特になったり、思わぬ影響が出たりするため可能であれば避けたいところだ

## 本題: SWRのmiddlewareを利用する

ここから本題だ。[SWRのmiddleware](https://swr.vercel.app/docs/middleware)を利用してmiddlewareからmockを返すようにする。

```tsx
test("SWRのmiddlewareを利用したパターン", () => {
  const testMiddleware: Middleware = () => {
    return (): SWRResponse<any, any> => {
      const mockData: any = {
        mock: "data"
      }
      return {
        data: mockData,
        error: undefined,
        mutate: (_) => Promise.resolve(),
        isValidating: false
      }
    }
  }
  const tree = renderer.create(
    <SWRConfig value={{ use: [testMiddleware] }}>
      <SearchResult word="dog" />
    </SWRConfig>
  ).toJSON()
  expect(tree).toMatchSnapshot()
})

// 結果:
// <div>
//   <pre>
//     <code>
//       {
//   "mock": "data"
// }
//     </code>
//   </pre>
// </div>
// `;
```

エラーケースをテストしたければ下記のようにもできる

```tsx
test("SWRからエラーが帰ってくる場合", () => {
  const testMiddleware: Middleware = () => {
    return (): SWRResponse<any, any> => {
      return {
        data: undefined,
        error: Error(),
        mutate: (_) => Promise.resolve(),
        isValidating: false
      }
    }
  }
  const tree = renderer.create(
    <SWRConfig value={{ use: [testMiddleware] }}>
      <SearchResult word="dog" />
    </SWRConfig>
  ).toJSON()
  expect(tree).toMatchSnapshot()
})

// 結果
// <div>
//   エラーが発生しました
// </div>
// `;

```

これでmiddlewareでテストをカバーすることができた。
上記例ではかなり単純化しているが、例えばキーによって返すデータを変更するなども可能だろう

```tsx
test("キーによって結果が変わる場合", () => {
  const testMiddleware: Middleware = () => {
    return (key: string): SWRResponse<any, any> => {
      const mockData = {
        "https://api.dictionaryapi.dev/api/v2/entries/en/dog": {
          descrption: "dog is kawaii"
        },
        "https://api.dictionaryapi.dev/api/v2/entries/en/meat": {
          descrption: "niku is umai"
        }
      }
      return {
        data: mockData?.[key],
        error: undefined,
        mutate: (_) => Promise.resolve(),
        isValidating: false
      }
    }
  }
  const tree = renderer.create(
    <SWRConfig value={{ use: [testMiddleware] }}>
      <SearchResult word="dog" />
      <SearchResult word="meat" />
    </SWRConfig>
  ).toJSON()
  expect(tree).toMatchSnapshot()
})

// 結果
// Array [
//   <div>
//     <pre>
//       <code>
//         {
//   "descrption": "dog is kawaii"
// }
//       </code>
//     </pre>
//   </div>,
//   <div>
//     <pre>
//       <code>
//         {
//   "descrption": "niku is umai"
// }
//       </code>
//     </pre>
//   </div>,
// ]
```