---
title: eslintのルールをVSCodeでだけerrorをwarningにする
emoji: 🚸
type: tech
topics:
  - eslint
  - typescript
published: true
---

ESLintに新たに[Bulk Suppression](https://eslint.org/docs/latest/use/suppressions)という機能が追加された。
これを実行すると、`eslint-suppressions.json`に無視するルールと件数が記録される

* 既存コードにはコマンドラインでエラーにならない
* 新規コードや、既存コードを編集するとエラーが出る
* VSCodeなどのエディタ上ではエラーとして表示される(まだプラグインが未対応なだけかも？)

これによって、エディタ上では認識出来たり、`eslint-suppressions.json`に意図しない変更があるかを確認出来るなど嬉しいことが多い。

## 課題: TypeScriptエラーとの区別がつかない

Bulk Suppressionによって、これまで躊躇していたエラーも追加しやすくなり、AIとのコーディングを考えると、自分が意識してなかったルールも追加したかったりとするものの、TypeScriptのエラーと混在されると困る課題がある。

エディタ上での視覚的な区別ができればよかったが、下線色をわけるなどは出来なさそうだったので、VSCodeでだけESLintのerrorをwarningに変換するアプローチにすることにした。

## VSCodeでだけerrorをwarning化する実装

`eslin.config.ts`で下記のようなコードで変換出来る（Flat Configの場合で記載）

```ts
const isOffSetting = (setting: unknown): boolean => {
  if (Array.isArray(setting)) {
    return setting[0] === "off" || setting[0] === 0
  }
  return setting === "off" || setting === 0
}

const convertEditorConfig = (configs: TSESLint.FlatConfig.Config[]): TSESLint.FlatConfig.Config[] => {
  const processRules = (rules: Rules): Rules => {
    if (!rules) return rules
    return Object.fromEntries(
      Object.entries(rules).map(([rule, setting]) => {
        if (isOffSetting(setting)) {
          return [rule, setting]
        }
        if (Array.isArray(setting)) {
          return [rule, ["warn", ...setting.slice(1)]]
        }
        return [rule, "warn"]
      })
    )
  }

  return configs.map((config) => {
    const convertRules = processRules(config.rules)
    return {
      ...config,
      ...(convertRules ? {
        rules: convertRules
      } : {}),
    }
  })
}
```

この関数を使って、VSCode CLIからの実行時だけ設定を変更する。
VSCodeから起動の場合、`process.env.VSCODE_CLI`が設定されているので、これが利用できる

```ts
const convertFlatConfig = (flatConfig) => {
  if (process.env.VSCODE_CLI === "1") {
    const convertConfig = convertEditorConfig(flatConfigs)
    return convertConfig
  }
  return flatConfig
}
```

これでVSCode上ではerrorがwarningとして表示され、TypeScriptのエラーと視覚的に区別できる。CI環境では通常のerrorのままなので、厳格なチェックを維持できる。