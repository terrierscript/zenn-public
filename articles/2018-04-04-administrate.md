---
title: administrateの小ネタ3つ
emoji: 🟤
type: tech
topics:
  - rails
  - ruby
  - administrate
published: true
terrier_export_from:
  - https://blog.terrier.dev/blog/2018/20180404000000-administrate-gem-namespace-generate/
  - https://blog.terrier.dev/blog/2018/20180405000000-administrate-spec-example_app/
  - https://blog.terrier.dev/blog/2018/20180405000000-administrate-new_resource/
---

> 記事作成時期:2018/04

# namespace付きでgenerateする
administrateで複数のパスを利用したい場合、namspaceによって切り分けたい。

ドキュメントに明記されてないが、こんな感じで引数オプションが存在している

```bash
$ rails generate administrate:install --namespace manager
$ rails generate administrate:dashboard --namespace manager
```


[administrate/pull/956](https://github.com/thoughtbot/administrate/pull/956)

# サンプルは/spec/example_appを見ると良い

administrateを使う時、下記に色々入っているため、参考にすると良い

https://github.com/thoughtbot/administrate/tree/master/spec/example_app

動いてるサンプルはこっち
http://administrate-prototype.herokuapp.com/admin

* 表示・編集
* read only機能
* Enum的なselect
* belongs_toなデータの扱い

などがある

# 新規リソース追加時new_resourceに、既存データをコピーして使う

何かlog的なデータで、最新のデータを使いまわして上書きしたい時のパターン
今のところcontrollerを生やして上書きするしか無さそう

---

::: message
https://github.com/thoughtbot/administrate/pull/1097/files
こちらがリリースされているので、バージョンによってはnewをcontrollerに生やさなくても良くなる可能性が高い
:::

```ruby
module Admin
  class SomeController < Admin::ApplicationController
    def new
      resource = new_resource
      authorize_resource(resource)
      render locals: {
        page: Administrate::Page::Form.new(dashboard, resource),
      }
    end

    private

    def new_resource
      # 最新データ
      copy = resource_class.order(created_at: :desc).first
      return copy if copy.present?     
      resource_class.new
    end
  end
end
```

