# 12章 Summary

- パスワードの再設定は Active Recordオブジェクトではないが、セッションやアカウント有効化の場合と同様に、リソースでモデル化できる
- Railsは、メール送信で扱うAction Mailerのアクションとビューを生成することができる
- Action MailerではテキストメールとHTMLメールの両方を利用できる
- メイラーアクションで定義したインスタンス変数は、他のアクションやビューと同様、メイラーのビューから参照できる
- パスワードを再設定させるために、生成したトークンを使って一意のURLを作る
- より安全なパスワード再設定のために、ハッシュ化したトークン (ダイジェスト) を使う
- メイラーのテストと統合テストは、どちらもUserメイラーの振舞いを確認するのに有用
- SendGridを使うとproduction環境からメールを送信できる


# パスワード再設定　実装の流れ
1. ユーザーがパスワードの再設定をリクエストすると、ユーザーが送信したメールアドレスをキーにしてデータベースからユーザーを見つける
2. 該当のメールアドレスがデータベースにある場合は、再設定用トークンとそれに対応するリセットダイジェストを生成する
3. 再設定用ダイジェストはデータベースに保存しておき、再設定用トークンはメールアドレスと一緒に、ユーザーに送信する有効化用メールのリンクに仕込んでおく
4. ユーザーがメールのリンクをクリックしたら、メールアドレスをキーとしてユーザーを探し、データベース内に保存しておいた再設定用ダイジェストと比較する (トークンを認証する)
5. 認証に成功したら、パスワード変更用のフォームをユーザーに表示する 


## NoMethodError

### 現状
エラーメッセージ
```
undefined method `reset_sent_at=' 
Did you mean? reset_token=

    self.reset_token = User.new_token
    update_attribute(:reset_digest, User.digest(reset_token))
    update_attribute(:reset_sent_at, Time.zone.now)
  end

  # パスワード再設定のメールを送信する

```

- ```reset_digest```だけ正常に生成されてる
- 上記コードを再度叩いてもう一度マイグレーション するとconflictしちゃう


### 原因
マイグレーション生成時のtypoでした！！ 
( \ の前のスペース入れ忘れてた）

正しいコード
```rails g migration add_reset_to_users reset_digest:string \ reset_sent_at:datetime```


### 対策 
db/migrate/[TimeStamp]_add_reset_to_users を直接修正して解決しました！

1. db/migrate/[TimeStamp]_add_reset_to_users を修正

エラーのコード
```rb: db/migrate/[TimeStamp]_add_reset_to_users
class AddResetToUsers < ActiveRecord::Migration[5.1]
  def change
    add_column :users, :reset_digest, :stringreset_sent_at
  end
end
```

修正したコード
```rb: db/migrate/[TimeStamp]_add_reset_to_users
class AddResetToUsers < ActiveRecord::Migration[5.1]
  def change
    add_column :users, :reset_digest, :string
    add_column :users, :reset_sent_at, :datetime
  end
end
```

2. 重複してるmigrationファイルがある場合は削除
3. 最後に```rails db:migrate``` も忘れずに！

参考記事:
- [rails generate migrationコマンドまとめ](https://qiita.com/zaru/items/cde2c46b6126867a1a64)
- [超初心者がRuby on Railsで詰まったところ](http://ex-agent.com/2018/03/28/rails_error/)
- [モデル生成用マイグレーションファイル作成時のコンフリクトエラーへの対処方法](https://qiita.com/maecha/items/fc150d73648b13a69b12)


## パスワード再設定フォームの面倒な問題
パスワード再設定メールに含まれるリンクを機能させるために、（再設定を実行するための）パスワード再設定フォームを表示するビューが必要

メールアドレスをキーとしてユーザーを検索する為にeditアクションとupdateアクションの両方でメールアドレスが必要

メールアドレス入りリンクのおかげで、editアクションでメールアドレスを取り出すことは問題ありません。
しかしフォームを一度送信してしまうと、この情報は消えてしまいます
b/c 再設定を実行するためのフォームにはパスワード入力フィールドと確認用フィールドしか作っていない為 
### 対策手順
- 隠しフィールドを作ってメールアドレスを保存
- フォームの情報を送信するときにメールアドレスの情報も一緒に送信できる


## 異なるフォームタグヘルパー

```f.hidden_field :email, @user.email```

これだとparams[:user][:email] にメールアドレスが保存される

form_withやform_forで渡すインスタンスがある場合(もしくはそれらのヘルパーを使っている場合)は、f.hidden_fieldを使う

今回はparams[:email]にメールアドレスを保存させる↓

```hidden_field_tag :email, @user.email```

一個だけパラメータを他のアクションへ単体で渡したい時は、hidden_field_tagを使う

c.f. [f.hidden_fieldとhidden_field_tagの使い方【Ruby on Rails】](https://sakurawi.hateblo.jp/entry/hidden_field)


## PasswordResetsコントローラのeditアクション
params[:email]のメールアドレスに対応するユーザーを保存するための@user変数を定義する 続けて、params[:id]の再設定用トークンとauthenticated?メソッドを使って、このユーザーが正当なユーザーであることを確認する 正当なユーザー＝(ユーザーが存在する、有効化されている、認証済みである） 
editアクションとupdateアクションのどちらの場合も正当な@userが存在する必要があるので、いくつかのbeforeフィルタを使って@userの検索とバリデーションを行います


# 12.3.2 パスワードを更新する
AccountActivationsコントローラのeditアクションでは、ユーザーの有効化ステータスをfalseからtrueに変更しましたが、今回の場合はフォームから新しいパスワードを送信するようになっています。したがって、フォームからの送信に対応するupdateアクションが必要になります。

1. このupdateアクションで考慮すること
パスワード再設定の有効期限が切れていないか
	editとupdateアクションに次のようなメソッドとbeforeフィルターを用意することで対応

2. 無効なパスワードであれば失敗させる (失敗した理由も表示する)
	更新が失敗したときにeditのビューが再描画され、リスト 12.14のパーシャルにエラーメッセージが表示されるようにすれば解決できます。

3. 新しいパスワードが空文字列になっていないか (ユーザー情報の編集ではOKだった)
	???

4. 新しいパスワードが正しければ、更新する
	更新が成功したときにパスワードを再設定し、あとはログインに成功したとき (リスト 8.25) と同様の処理を進めていけば問題なさそうです。


## 新しいパスワードが空文字列になっていないか
Userモデルではパスワードは空でもよいという実装をしている(リスト10.13)
しかし再設定ではパスワードのフィールドが空では再設定にならないので
明示的に（パスワードのフィールドが空であることを）キャッチするコードを追加する！
（確認フィールドが空の場合は、確認フィールドのバリデーションで検出されてエラーメッセージが表示される）
→@userオブジェクトにエラーメッセージを追加

```@user.errors.add(:password, :blank)```
オブジェクト.errors.add(対象のカラム, ‘エラーの内容’)
エラーの内容にblankオプションを指定することでI18nで多言語している場合でも言語に合わせた適切なメッセージを表示してくれる


## password_reset_expired?メソッドの実装
password_reset_expired?メソッドをUserモデルで定義

```reset_sent_at < 2.hours.ago```: 
パスワード再設定の期限を設定して、2時間以上パスワードが再設定されなかった場合は期限切れとする処理


## この時点でtest red になる！ (fixed)

error message:
```
FAIL["test_should_get_edit", PasswordResetsControllerTest, 1.2200050550000014]
 test_should_get_edit#PasswordResetsControllerTest (1.22s)
        Expected response to be a <2XX: success>, but was a <302: Found> redirect to <http://www.example.com/>
        Response body: <html><body>You are being <a href="http://www.example.com/">redirected</a>.</body></html>
        test/controllers/password_resets_controller_test.rb:11:in `block in <class:PasswordResetsControllerTest>'
```

### 原因
余計なテストを作成していた

### 対策
不要なテストの削除
```rm test/controllers/password_resets_controller_test.rb```

cf. [Railsチュートリアルの12章で統合テスト実行時に302: Foundが出る場合の対処法](https://qiita.com/yokoyan/items/5ecbb6ed5bfb157783dc)


## エディタでwarning が出る (Cloud9)
warning message:
warning: ambiguous first argument; put parentheses or even spaces
```assert_match /expired/i, response.body```

指示通り、かっこやスペースを入れてもエラー

```assert_match / expired /i, response.body```
error message:
syntax error, unexpected end-of-input, expecting end


### 原因
参考ページによると、サポートされていないっぽい


### 対策
なし（warning が出続けてて気持ち悪いけど、そのまま進めます）


cf. 
- [What triggers the Ruby warning about ambiguous first argument?](https://stackoverflow.com/questions/5239805/what-triggers-the-ruby-warning-about-ambiguous-first-argument)
- [Syntax error, unexpected tREGEXP_BEG #444](https://github.com/mruby/mruby/issues/444)


