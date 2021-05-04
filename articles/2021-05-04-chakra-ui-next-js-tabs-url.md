---
title: Chakra UIのTabsをnext.jsでURL同期させる
emoji: 🥘
type: tech
topics:
  - nextjs
  - chakraui
  - javascript
  - typescript
published: true
---

Chakra UIのTabsを利用した際、このタブ切り替えとURLを同期したくなったので、next.jsとの組み合わせでまとめる

ファイル名は`/tabs/[[...path]].tsx`として、パスをすべて拾うようにする。[^1]

[^1]: `[[...path]]`については [この記事](https://zenn.dev/terrierscript/articles/2021-04-29-next-js-props-catch-all-routes)にまとめたので参照のこと

```tsx
/* /tabs/[[...path]].tsx */

import { Box, Tab, TabList, TabPanel, TabPanels, Tabs } from "@chakra-ui/react"
import React, { FC } from "react"
import { useRouter } from "next/router"
import { GetServerSideProps } from "next"
const tabMap = [
  "dog", "cat"
]
const Page: FC<{ path: string }> = ({ path }) => {
  const initialTab = Math.max(tabMap.indexOf(path), 0) // path値から初期タブを決定
  const router = useRouter()

  return <Box>
    <Tabs
      onChange={(idx) => {
        // タブが変更されたらrouterへpush。
        router.push({
          pathname: router.pathname,
          query: {
            path: [tabMap[idx]]
          }
        }, undefined, { shallow: true }) // shallowすることでrouterを再度呼び出さない
      }}
      defaultIndex={initialTab}
    >
      <TabList>
        <Tab>
          {tabMap[0]}
        </Tab>
        <Tab>
          {tabMap[1]}
        </Tab>
      </TabList>
      <TabPanels>
        <TabPanel>
          🐶
        </TabPanel>
        <TabPanel>
          🐱
        </TabPanel>
      </TabPanels>
    </Tabs>
  </Box>
}

export const getServerSideProps: GetServerSideProps = async (req) => {
  const path = req.query.path?.[0] ?? null
  return {
    props: { path }
  }
}

export default Page
```

* `useRouter`からpathを取得するなどでも良かったが、`getServerSideProps`からパスを取得するようにした。
* onChangeで変更があった際にshallowでpushする
* nextjs以外なら、`history.pushstate`を使うことになるだろう