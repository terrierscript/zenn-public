---
title: ブラウザ上でちょっとした効果音をノーライブラリで鳴らす
emoji: 🔊
type: tech
topics:
  - audio
  - typesript
  - react
published: true
---

ブラウザでちょっとしたSEみたいなものを鳴らしたいケースが度々ある。

今であれば[AudioContext](https://developer.mozilla.org/ja/docs/Web/API/AudioContext)と[OscillatorNode](https://developer.mozilla.org/ja/docs/Web/API/OscillatorNode)で十分出来てしまったのでメモ。

なお、今回の方法だと微妙にノイズっぽくなったりするので、本格的なものなら[Tone.js](https://github.com/Tonejs/Tone.js)を利用するのが良い。

## デモ

@[codesandbox](https://codesandbox.io/embed/eloquent-mountain-o1jqmw?fontsize=14&theme=dark)

# 解説

## AudioContext + OscillatorNodeの基礎
音を鳴らす部分はざっくりこのあたり。デバッグコンソールでも下記程度で動くのが確認出来るはずだ。

```ts
const audioCtx = new window.AudioContext();
const oscillator = audioCtx.createOscillator();

oscillator.type = "sine"; 
oscillator.frequency.setValueAtTime(1200, audioCtx.currentTime);
oscillator.connect(audioCtx.destination);
oscillator.start(audioCtx.currentTime);
oscillator.stop(audioCtx.currentTime + 0.5);
```

## 短い無音を出してサウンド処理の準備をする

昨今のブラウザでは、クリックイベントなどを挟まないとAudio APIを利用することが出来ない。

これを回避するのに無音を一瞬出すコンポーネントを挟む

```tsx
const SoundReady: FC<PropsWithChildren<{}>> = ({ children }) => {
  const [ready, setReady] = useState(false)
  // 無音を再生。
  const playReadySound = (onPlayEnd: () => void) => {
    const audioCtx = new window.AudioContext()
    const oscillator = audioCtx.createOscillator()
    oscillator.type = 'triangle'
    oscillator.frequency.setValueAtTime(0, audioCtx.currentTime)
    oscillator.connect(audioCtx.destination)

    oscillator.onended = () => {
      onPlayEnd()
    }
    oscillator.start(0)
    oscillator.stop(0)
  }

  const onSetup = () => {
    playReadySound(() => {
      setReady(true)
    })
  }

  if (!ready) {
    return <Button onClick={() => onSetup()}>
      Setup
    </Button>
  }

  return <>{children}</>
}
```

## ピポッという音を鳴らす

ここからはイベントに合わせて音を鳴らすので、その部分を組み立てていく。

充電が無くなってきた時のような「ピポッ」というような音を作るなら、下記のように`frequency`を時間ごとに変えるような音にすると出来る。

```ts
  const onPress = () => {
    const audioCtx = new window.AudioContext()

    const oscillator = audioCtx.createOscillator()
    oscillator.type = 'sine'
    oscillator.frequency.setValueAtTime(1200, audioCtx.currentTime)
    oscillator.frequency.setValueAtTime(800, audioCtx.currentTime + 0.1)
    oscillator.connect(audioCtx.destination)
    oscillator.start(audioCtx.currentTime)
    oscillator.stop(audioCtx.currentTime + 0.2)
  }

```

## ピピッという音を鳴らす

電子マネーっぽい「ピピッ」という音なら下記のように２つのNodeを作成してそれぞれに設定をするとそれっぽくなる。

```ts
  const onPress = () => {
    const audioCtx = new window.AudioContext()

    const nodes = [
      audioCtx.createOscillator(),
      audioCtx.createOscillator()
    ]
    const hz = 1700
    nodes.map(node => {
      node.type = 'sine'
      node.frequency.setValueAtTime(hz, audioCtx.currentTime)
      node.connect(audioCtx.destination)
    })

    const length = 0.1
    const rest = 0.025
    nodes[0].start(audioCtx.currentTime)
    nodes[0].stop(audioCtx.currentTime + length)
    nodes[1].start(audioCtx.currentTime + length + rest)
    nodes[1].stop(audioCtx.currentTime + length * 2 + rest)
  }
```

## 押しっぱなしにしてる間鳴るボタン

最後に「ブー」と押してる間鳴るボタン。
こちらはstartとstopを分離する必要があるので、今回は`useRef`を利用する。

```tsx
const SoundBoo = () => {
  const audioCtxRef = useRef<AudioContext>()
  const oscillatorRef = useRef<OscillatorNode>()
  useEffect(() => {
    audioCtxRef.current = new AudioContext()
  }, [])
  const start = () => {
    if (!audioCtxRef.current || oscillatorRef.current) {
      return
    }
    const audioCtx = audioCtxRef.current
    const oscillator = audioCtx.createOscillator()
    oscillator.type = 'sawtooth'
    oscillator.frequency.setValueAtTime(100, audioCtx.currentTime)
    oscillator.connect(audioCtx.destination)
    oscillatorRef.current = oscillator
    oscillator.start(audioCtx.currentTime)
  }
  const stop = () => {
    if (!audioCtxRef.current) {
      return
    }
    oscillatorRef.current?.stop(audioCtxRef.current.currentTime)
    oscillatorRef.current?.disconnect()
    oscillatorRef.current = undefined
  }

  return <Stack>
    <Button
      onMouseDown={() => start()}
      onMouseUp={() => stop()}
      onMouseOut={() => stop()}
      colorScheme={"red"} variant="outline">
      ブー ❌
    </Button>
  </Stack>
}
```

`onMouseUp`だけだと、押しっぱなしにしたままマウスが外に出てしまった場合に音が鳴りっぱなしになってしまうので、`onMouseOut`にもつける
