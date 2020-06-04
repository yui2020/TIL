# 11章 Summary
- アカウント有効化は Active Recordオブジェクトではないが、セッションの場合と同様に、リソースでモデル化できる
- Railsは、メール送信で扱うAction Mailerのアクションとビューを生成することができる
- Action MailerではテキストメールとHTMLメールの両方を利用できる
- メイラーアクションで定義したインスタンス変数は、他のアクションやビューと同様、メイラーのビューから参照できる
- アカウントを有効化させるために、生成したトークンを使って一意のURLを作る
- より安全なアカウント有効化のために、ハッシュ化したトークン (ダイジェスト) を使う
- メイラーのテストと統合テストは、どちらもUserメイラーの振舞いを確認するのに有用
- SendGridを使うと、production環境からメールを送信できる

# アカウントの有効化
アカウントを有効化する段取りは、

1. ユーザーの初期状態は「有効化されていない」(unactivated) にしておく。
2. ユーザー登録が行われたときに、有効化トークンと、それに対応する有効化ダイジェストを生成する。
3. 有効化ダイジェストはデータベースに保存しておき、有効化トークンはメールアドレスと一緒に、ユーザーに送信する有効化用メールのリンクに仕込んでおく3 。
4. ユーザーがメールのリンクをクリックしたら、アプリケーションはメールアドレスをキーにしてユーザーを探し、データベース内に保存しておいた有効化ダイジェストと比較することでトークンを認証する。
5. ユーザーを認証できたら、ユーザーのステータスを「有効化されていない」から「有効化済み」(activated) に変更する。


## before_createコールバック
オブジェクトが作成されたとき(create) だけコールバックを呼び出す 
= それ以外のときには呼び出したくない

e.g. ```before_create :create_activation_digest```

上のコードはメソッド参照と呼ばれる
こうするとRailsはcreate_activation_digestというメソッドを探し、ユーザーを作成する前に実行する

## rememberメソッドとの違い
コールバックに関連付けるメソッドは記憶トークンや記憶ダイジェストのために作ったメソッドを使いまわしてるけどもちろん違いもある！

```rb
# 永続セッションのためにユーザーをデータベースに記憶する
def remember
  self.remember_token = User.new_token
  update_attribute(:remember_digest, User.digest(remember_token))
end

# 有効化トークンとダイジェストを作成および代入する
def create_activation_digest
  self.activation_token  = User.new_token
  self.activation_digest = User.digest(activation_token)
end
```

記憶トークンやダイジェストは既にデータベースにいるユーザーのために作成されるのでupdate_attributeを使って情報を更新・保存しているが
before_createコールバックの方はユーザーが作成される前に呼び出されることなので更新される属性がない→新しく取得


## Time.zone.now
Railsの組み込みヘルパーであり、サーバーのタイムゾーンに応じたタイムスタンプを返します。


# 11.2 アカウント有効化のメール送信
Action Mailerライブラリを使ってUserのメイラーを追加します。
このメイラーはUsersコントローラのcreateアクションで有効化リンクをメール送信するために使います。


## Action Mailerライブラリとな
Railsを使ってメールを送信する仕組み（雑） アクションやviewからメールを送信できるようになる メイラーの動作はコントローラのアクションと似ているしテンプレートはviewに似ている 詳しくはこちらとか→Railsガイド｜Action Mailer の基礎


## リスト 11.6: Userメイラーの生成
```$ rails generate mailer UserMailer account_activation password_reset```

account_activationメソッドとpassword_resetメソッドが生成される


## viewに含めるコード
ユーザー名を含む挨拶文と有効化リンクを追加する。
→Railsサーバーでユーザーをメールアドレスで検索して有効化トークンを認証できるようにするため、リンクにはメールアドレスとトークンを両方含めておく必要があります

```rb
#このコードを考える
edit_account_activation_url(@user.activation_token, ...)

#このメソッドは
edit_user_url(user)
#↓このURLを生成
http://www.example.com/users/1/edit
#これに対応するアカウント有効化リンクのベースURLは↓
#ランダム文字列はnew_tokenメソッドで生成されたもの
http://www.example.com/account_activations/q5lt38hQDc_959PVoo6b7A/edit

#このコードを考える
edit_account_activation_url(@user.activation_token, ...)
 
#このメソッドは
edit_user_url(user)
#↓このURLを生成
http://www.example.com/users/1/edit
#これに対応するアカウント有効化リンクのベースURLは↓
#ランダム文字列はnew_tokenメソッドで生成されたもの
http://www.example.com/account_activations/q5lt38hQDc_959PVoo6b7A/edit
```

URLで使えるようにBase64でエンコードされています。
これはちょうど/users/1/editの「1」のようなユーザーIDと同じ役割を果たします。
このトークンは、特にAccountActivationsコントローラのeditアクションではparamsハッシュでparams[:id]として参照できます。

さらにクエリパラメータを使ってメールアドレスを組み込む

```account_activations/q5lt38hQDc_959PVoo6b7A/edit?email=foo%40example.com```


## クエリパラメータとは
クエリパラメータとは、URLの末尾で疑問符「?」に続けてキーと値のペアを記述した

上記で言えばedit?email=foo%40example.comの部分
edit?に続けてキー「email」値「foo%40example.com」を呼んでいる
「%40」は「＠」のエスケープなので実際呼び出しているのは「foo@example.com」
Railsでクエリパラメータを設定するには、名前付きルートに対して次のようなハッシュを追加する

```edit_account_activation_url(@user.activation_token, email: @user.email)```

こうすることでRailsが特殊な文字を自動的にエスケープしてくれるし、コントローラでparams[:email]からメールアドレスを取り出すときには、自動的にエスケープを解除してくれる。優しい


## クラウドIDEホスト名がわからない！
解決策: [rails チュートリアルの１１章の２・２にてクラウドIDEのホスト名がわからない](https://teratail.com/questions/125713)

解決しました！

## メールプレビューが開けない
```NoMethodError in Rails::MailersController#preview
undefined method `activation_token=' for nil:NilClass```

解決策: [Rails チュートリアル第4版　11.2.2 送信メールのプレビューでエラー](https://qiita.com/muttsu__/items/693010299a54884b473d)

解決！よっしゃー！！

原因: 
```
rails db:migrate:reset
rails db:seed
```
を実行した時にサーバー開いたままだったので、うまくいってなかったっぽい


## assert_matchメソッド
正規表現で文字列をテストできます。

e.g. 
```rb
assert_match 'foo', 'foobar'      # true
assert_match 'baz', 'foobar'      # false
assert_match /\w+/, 'foobar'      # true
assert_match /\w+/, '$#!*+@'      # false
```

# メタプログラミング
「プログラムでプログラムを作成する」ことです。

## sendメソッド
```
$ rails console
>> a = [1, 2, 3]
>> a.length
=> 3
>> a.send(:length) # = a.length
=> 3
>> a.send("length") # = a.length
=> 3
```

このときsendを通して渡したシンボル:lengthや文字列"length"は、いずれもlengthメソッドと同じ結果になりました。
= どちらもオブジェクトにlengthメソッドを渡しているため、等価なのです。

データベースの最初のユーザーが持つactivation_digest属性にアクセスする例です。

```
>> user = User.first
>> user.activation_digest
=> "$2a$10$4e6TFzEJAVNyjLv8Q5u22ensMt28qEkx0roaZvtRcp6UZKRM6N9Ae"
>> user.send(:activation_digest)
=> "$2a$10$4e6TFzEJAVNyjLv8Q5u22ensMt28qEkx0roaZvtRcp6UZKRM6N9Ae"
>> user.send("activation_digest")
=> "$2a$10$4e6TFzEJAVNyjLv8Q5u22ensMt28qEkx0roaZvtRcp6UZKRM6N9Ae"
>> attribute = :activation
>> user.send("#{attribute}_digest")
=> "$2a$10$4e6TFzEJAVNyjLv8Q5u22ensMt28qEkx0roaZvtRcp6UZKRM6N9Ae"
```

最後の例では、
1. シンボル:activationと等しいattribute変数を定義
2. 文字列の式展開 (interpolation) を使って引数を正しく組み立て
3. sendに渡す

文字列'activation'でも同じことができますが、Rubyではシンボルを使う方が一般的です。

シンボルと文字列どちらを使った場合でも、上のコードは次のように文字列に変換されます。
```”"#{attribute}_digest”``` = ```activation_digest”```

## コード解説　authenticated?メソッドを書き換え
```rb
def authenticated?(remember_token)
  digest = self.send("remember_digest")
  return false if digest.nil?
  BCrypt::Password.new(digest).is_password?(remember_token)
end
```

上のコードの各引数を一般化し、文字列の式展開も利用すると、次のようなコードにできます。

```rb
def authenticated?(attribute, token)
  digest = self.send("#{attribute}_digest")
  return false if digest.nil?
  BCrypt::Password.new(digest).is_password?(token)
end
```

他の認証でも使えるように、上では2番目の引数tokenの名前を変更して一般化している点に注意してください。

また、このコードはモデル内にあるのでselfは省略することもできます。最終的にRubyらしく書かれたコードは、次のようになります。

```rb
def authenticated?(attribute, token)
  digest = send("#{attribute}_digest")
  return false if digest.nil?
  BCrypt::Password.new(digest).is_password?(token)
end
```


# 11.3.2 editアクションで有効化

```	if user && !user.activated? && user.authenticated?(:activation, params[:id])```

!user.activated?というコードは、既に有効になっているユーザーを誤って再度有効化しないために必要です。

正当であろうとなかろうと、有効化が行われるとユーザーはログイン状態になります。
もしこのコードがなければ、攻撃者がユーザーの有効化リンクを後から盗みだしてクリックするだけで、本当のユーザーとしてログインできてしまいます。そうした攻撃を防ぐためにこのコードは非常に重要です。


```assert_equal 1, ActionMailer::Base.deliveries.size```
上のコードは、配信されたメッセージがきっかり1つであるかどうかを確認します


 ## assignsメソッド
対応するアクション内のインスタンス変数にアクセスできるようになります。
例えば、Usersコントローラのcreateアクションでは@userというインスタンス変数が定義されていますが (リスト 11.23)、テストでassigns(:user)と書くとこのインスタンス変数にアクセスできるようになる


## 消えたuser.
Userモデルには「user」という変数はないので、コントローラのコードをそのまま移すとエラーになる

```rb
user.update_attribute(:activated,    true)
user.update_attribute(:activated_at, Time.zone.now)

#これ↑がこう↓なる

update_attribute(:activated,    true)
update_attribute(:activated_at, Time.zone.now)
```

```rb
 
update_attribute(:activated,    true)
update_attribute(:activated_at, Time.zone.now)
(userをselfに切り替えるという手もあるのですが、selfはモデル内では必須ではないと6.2.5で解説したことを思い出しましょう。) Userメイラー内の呼び出しでは、@userがselfに変更されている点にもご注目ください。

```rb
#なので
UserMailer.account_activation(@user).deliver_now

#これ↑がこう↓こうなる

UserMailer.account_activation(self).deliver_now
```
 
どんなに簡単なリファクタリングであっても、この手の変更はつい忘れてしまうものです。
テストをきちんと書いておけば、この種の見落としを検出できます。


### update_columns
データベースに直接アクセスしてカラム（属性）の値を更新する
バリデーションやコールバックはスキップされる


# 11.4 本番環境でのメール送信
本番環境からメール送信するために、「SendGrid」というHerokuアドオンを利用してアカウントを検証します

