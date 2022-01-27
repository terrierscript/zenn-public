---
title: NativeBaseでContrast警告が誤判定表示されるのを解決する
emoji: 🌓
type: tech
topics:
  - reactnative
  - nativebase
  - react
  - typescript
published: true
---

nativebaseで下記のような警告が出るケースがある。

```
NativeBase: The contrast ratio of 1.4434776958677475:1 for 
primary.500 on transparent
```

これはコントラストについての警告しており、基本的にはthemeの`primary.500`を調整するなどが妥当ではある。

しかしコントラストの計算が[このような関数](https://github.com/GeekyAnts/NativeBase/blob/b4ed987dd88bc1da45bdeb909c974373ec7b062d/src/hooks/useContrastText.ts#L118-L122)でされており、背景が`transparent`になっている場合、`#000000`（黒）と同等に扱われて計算されてしまい実態と違う警告がされてしまうことがある。

具体例としては明るめの背景の上にvariant="link"など透過される場合などで適切でない警告が発生してしまう

```tsx
<Box bg="white">
  <Button variant="link">Send</Button>
</Box>
```

### 解決策

* https://github.com/GeekyAnts/NativeBase/issues/4345

今のところ値を調整する以外ではこのチェック自体をオフにするしか解決手段はなさそうだ。
あまり気持ちのいい解決ではないが、透過に対して計算しているので仕方がない部分にはなりそうだ

```tsx
<NativeBaseProvider 
  config={{
    suppressColorAccessibilityWarning: true
  }}
>
```
上記のように`suppressColorAccessibilityWarning`の値で抑止できる