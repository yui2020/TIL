# 6章 Summary
- マイグレーションを使うことで、アプリケーションのデータモデルを修正することができる
- Active Recordを使うと、データモデルを作成したり操作したりするための多数のメソッドが使えるようになる
- Active Recordのバリデーションを使うと、モデルに対して制限を追加することができる
- よくあるバリデーションには、存在性・長さ・フォーマットなどがある
- 正規表現は謎めいて見えるが非常に強力である
- データベースにインデックスを追加することで検索効率が向上する。また、データベースレベルでの一意性を保証するためにも使われる
- has_secure_passwordメソッドを使うことで、モデルに対してセキュアなパスワードを追加することができる

---
<br>

# Active Record
データベースとやりとりをするデフォルトのRailsライブラリ1 。Active Recordは、データオブジェクトの作成/保存/検索のためのメソッドを持つ

- コントローラ名 複数形 (e.g. Users)
- モデル名 単数形 (e.g. User)

# マイグレーション とは
- データベースに与える変更を定義したchangeメソッドの集まり
- ファイル名の頭にタイムスタンプが付く

## タイムスタンプ とは
そのデータの作成時刻・更新時刻を自動的に記録する
e.g. ```created_at```, ```updated_at```

## マジックカラム (Magic Columns)
使われる予定がある単語（予約語）なので変数名や関数名に定義できない

# Sandboxモードとは
コンソールの終了時にすべての変更を “ロールバック” (取り消し) する

## Sandbox の起動
```$ rails console --sandbox```

## update_attributesメソッド
- 属性のハッシュを受け取り、成功時には更新と保存を続けて同時に行う (保存に成功した場合はtrueを返す)
- ただし、検証に1つでも失敗すると、update_attributesの呼び出しは失敗
- 検証を回避するといった効果もある

# 検証 (Validation)
e.g.
- 存在性 (presence)の検証
- 長さ (length)の検証
- フォーマット (format)の検証
- 一意性 (uniqueness)の検証
 
### setupメソッド
setup内に書かれた処理は、各テストが走る直前に実行される

### rails test オプション
- ```rails test:integration```
- ```rails test:models```

c.f. [assert_not と valid? を使い validation が有効かどうか確認するテスト 6.2.2 存在性を検証する](https://qiita.com/kenchanayo/items/e5a661b0fa2f521072d4)

### Rails 省略記法
メソッドの最後の引数としてハッシュを渡す場合、波カッコを付けなくてもいい

e.g. 
- ```validates :name, presence: true``` i.e. ```validates(:name, presence: true)```

- validates(検証したい場所, 検証したい内容) i.e. validates(検証するカラム名, バリデーションヘルパー:  検証パラメータ)

## errorsオブジェクトを使って確認
```error.full_messages```: 起こっているエラーを用意されているメッセージで教えてくれる便利なチェーンメソッド

e.g.
```console: console
>> user = User.new(name: "", email: "mhartl@example.com")
>> user.valid?
=> false
>> user.errors.full_messages
=> ["Name can't be blank"]
```

### %w[]
文字列の配列を簡単に作れる

```console: console
>> %w[foo bar baz]
=> ["foo", "bar", "baz"]
```
## format ヘルパー
validates :email, format: { with: /正規表現/ }
withオプションで与えられた正規表現と属性の値がマッチするか検証するためのヘルパー

# 正規表現
```
/\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i
```
[Ruby on Rails Tutorial: 正規表現に関する解説ページ](https://railstutorial.jp/chapters/modeling_users?version=5.1#table-valid_email_regex)

## Rubular
対話的に正規表現を試せる
https://rubular.com/

### 定数の表記
大文字で始まる名前はRubyでは定数を意味する
e.g. VALID_EMAIL_REGEX: 定数

## dup メソッド
同じ属性を持つデータを複製するためのメソッド
e.g. ```@user.dup```

## case_sensitive
- uniqueness:ヘルパーのオプション
- 一意性制約で大文字小文字を区別するかどうかを指定するもの

e.g. ```case_sensitive: false```
uniqueness:値が一意であるかの検証に（大文字と小文字を区別しない）というオプションを追加している

## fixture とは
テストDB用のサンプルデータが含まれるファイル

# セキュアなパスワード
ユーザーの認証は、
1. パスワードの送信
2. ハッシュ化
3. データベース内のハッシュ化された値との比較
という手順で進んでいきます。

比較の結果が一致すれば、送信されたパスワードは正しいと認識され、そのユーザーは認証されます。

## ハッシュ化
- ハッシュ関数を使って、入力されたデータを元に戻せない (不可逆な) データにする処理
- ジャガイモをハッシュポテトにすると元のジャガイモには戻せない、みたいな

# has_secure_password メソッドを使う

- セキュアにハッシュ化したパスワードを、データベース内のpassword_digestという属性に保存できるようになる。
- 2つのペアの仮想的な属性 (passwordとpassword_confirmation) が使えるようになる。また、存在性と値が一致するかどうかのバリデーションも追加される18 。
- authenticateメソッドが使えるようになる (引数の文字列がパスワードと一致するとUserオブジェクトを、間違っているとfalseを返すメソッド) 。

- has_secure_passwordの機能を使えるようにするためにUserモデルに**password_digest属性を追加する**（passwordではないので注意！）

- has_secure_passwordを使う為に必要なgem ”bcrypt” をGemfileに追加
 
## authenticateメソッド
引数に渡された文字列 (パスワード) をハッシュ化した値と、データベース内にあるpassword_digestカラムの値を比較

### 論理値オブジェクトへの変換
!!でそのオブジェクトが対応する論理値オブジェクトに変換できる (4.2.3)

```console: console
>> !!user.authenticate("foobar")
=> true
```

### heroku 本番環境のコンソールに接続 (sandbox)
```$ heroku run rails console --sandbox```
