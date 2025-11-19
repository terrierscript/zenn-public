---
title: vitestのflakyテストをgithub actionsではたまに実行するようにする
emoji: ❄️
type: tech
topics: [vitest, typescript,github]
published: true
---

vitestでコケがちなflakyテストに対して、「捨てたくもないし存在は忘れたくないけど毎回コケると困る」みたいにする方法を考えた

# vitestの標準機能

残念ながら、vitestにおいて、tagのような機能は予定されてないようだった

- https://github.com/vitest-dev/vitest/issues/1057
- https://github.com/vitest-dev/vitest/issues/5346

そのため自前でどうにか管理する必要が出てきた

# どうするか？

下記３つがありそう

- A. `@flaky`のようにテスト名にマーキングする
- B. `runIf`を加工してflakyにする
- C. `Foo.flaky.test.ts`のようにファイルを分ける

Aの場合、特定のキーワードを無視するというのが若干複雑そうだったので除外した。

```js
"test:flaky": --testNamePattern="^(?!.*\@flaky).*$"
"test:stable": --testNamePattern="@flaky"
```

Bの場合は下記のような具合。ただこれだと「flakyを除外して実行」を考えるとちょっと厄介な部分が多く諦めた

```ts
const runFlaky = () => process.env.RUN_FLAKY_TEST
export const flakyTest = test.runIf(runFlaky())

flakyTest("....")
```

Cの場合は単にファイル名なので下記のような具合。
flakyな部分を別ファイルに切り出す面倒さはあるものの、管理のわかりやすさなど考えるとこれが一番良さそうだった。

```js
"test:flaky": "vitest run flaky.test.ts",
"test:stable": "vitest run --exclude '**/*.flaky.test.ts'",
```

# github acitonの設定

あとはこれをgithub actionsでたまに実行されるようにする。


```yml
- run: npm run test:stable
  name: Run stable tests
- id: decide_flaky
  run: echo "should_run=$(( $RANDOM % 10 < 2 ))" >> $GITHUB_OUTPUT
- run: npm run test:flaky
  name: Run flaky tests
  if: steps.decide_flaky.outputs.should_run == '1'
```

ワンライナーでやりたいところだったが、ちょっと難しそうなので、一回変数を格納してそれを利用する感じに落ち着いた
