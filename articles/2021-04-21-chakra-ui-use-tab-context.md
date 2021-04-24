---
title: Chakra UIにはuseTabsContextがある
emoji: 🗂
type: tech
topics:
  - chakraui
  - javascript
  - react
published: true
---

Chakra UIの[Tabs](https://chakra-ui.com/docs/disclosure/tabs)は、まだドキュメントされてないが。`useTabsContext`を使うことで、内部データにアクセスすることが出来るようだった。

* https://github.com/chakra-ui/chakra-ui/blob/117f2db20fbc7c261cdf9daa528a708eb191f6d6/packages/tabs/src/use-tabs.ts#L178

内部APIっぽくて使って良いのか若干心配ではあるが、[このPR](https://github.com/chakra-ui/chakra-ui/pull/3187)を見ると、外部利用されることを想定しているようだ。


### 使い方
例えば下記のような例を考えてみる

```jsx
const TabsSample = () => {
  return <Tabs>
    <TabList>
      <Tab>1</Tab>
      <Tab>2</Tab>
      <Tab>3</Tab>
    </TabList>
    <TabPanels>
      <TabPanel>1 panel</TabPanel>
      <TabPanel>2 panel</TabPanel>
      <TabPanel>3 panel</TabPanel>
    </TabPanels>
  </Tabs>
}
```

Contextを利用することになるので、まず`<Tabs>`とその子要素を分離しよう。


```jsx
const TabsInner = () => {
  return <>
    <TabList>
      <Tab>1</Tab>
      <Tab>2</Tab>
      <Tab>3</Tab>
    </TabList>
    <TabPanels>
      <TabPanel>1 panel</TabPanel>
      <TabPanel>2 panel</TabPanel>
      <TabPanel>3 panel</TabPanel>
    </TabPanels>
  </>
}

const TabsOuter = () => {
  // Tabs ≒ TabsProvider
  return <Tabs>
    <TabsInner />
  </Tabs>
}

```
`<Tabs>`自体は表示はなく、ほとんど`<TabsProvider>`と思っても差し支え無いだろう。[^1]
[^1]: ソースはこのへん: https://github.com/chakra-ui/chakra-ui/blob/117f2db20fbc7c261cdf9daa528a708eb191f6d6/packages/tabs/src/tabs.tsx#L46-L52

あとは`<TabsInner>`側でhooksを利用できる

```jsx
const TabsInner = () => {
  const { selectedIndex, focusedIndex } = useTabsContext()
  return <Box>
    <Box>{selectedIndex}</Box>
    <Box>{focusedIndex}</Box>
    <TabList>
      <Tab>1</Tab>
      <Tab>2</Tab>
      <Tab>3</Tab>
    </TabList>
    <TabPanels>
      <TabPanel>1 panel</TabPanel>
      <TabPanel>2 panel</TabPanel>
      <TabPanel>3 panel</TabPanel>
    </TabPanels>
  </Box>
}
```

まだundocumentedな上にあまり使う機会が多いものではないが、たまにレンダリングの制御関連で必要になったりするので、知っておいて損は無いだろう