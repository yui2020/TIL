# 7章 Summary
- debugメソッドを使うことで、役立つデバッグ情報を表示できる
- Sassのmixin機能を使うと、CSSのルールをまとめたり他の場所で再利用できるようになる
- Railsには標準で3つ環境が備わっており、それぞれ開発環境 (development)、テスト環境 (test)、本番環境 (production)と呼ぶ
- 標準的なRESTfulなURLを通して、ユーザー情報をリソースとして扱えるようになった
- Gravatarを使うと、ユーザーのプロフィール画像を簡単に表示できるようになる
- form_forヘルパーは、Active Recordのオブジェクトに対応したフォームを生成する
- ユーザー登録に失敗した場合はnewビューを再描画するようにした。その際、Active Recordが自動的に検知したエラーメッセージを表示できるようにした
- flash変数を使うと、一時的なメッセージを表示できるようになる
- ユーザー登録に成功すると、データベース上にユーザーが追加、プロフィールページにリダイレクト、ウェルカムメッセージの表示といった順で処理が進む
- 統合テストを使うことで送信フォームの振る舞いを検証したり、バグの発生を検知したりできる
- セキュアな通信と高いパフォーマンスを確保するために、本番環境ではSSLとPumaを導入した

# デバッグ情報を開発環境のみで表示する
```Ruby
<%= debug(params) if Rails.env.development? %>
```
```if Rails.env.development?``` : 開発環境 (development) のみで表示

### Rails の3つの環境
1. テスト環境 (test)
2. 開発環境 (development)
3. 本番環境 (production) 
[コラム7.1 Rails の3つの環境](https://railstutorial.jp/chapters/sign_up?version=5.1#aside-rails_environments)

## @mixin とは
まとめておいたコードを複数個所で簡単に呼び出すことでが出来る便利な機能
@mixin mixin名で定義して@include mixin名で呼び出す

## RESTアーキテクチャの復習
データの作成、表示、更新、削除をリソース (Resources) として扱うということ [コラム2.2](https://railstutorial.jp/chapters/toy_app?version=5.1#aside-REST) 

## resources: users
ユーザーのURLを生成するための多数の名前付きルート (5.3.3) と共に、RESTfulなUsersリソースで必要となるすべてのアクションが利用できるようになる

# debugger
- 今後Railsアプリケーションの中でよく分からない挙動があったら、debuggerを差し込んで調べみる
- トラブルが起こっていそうなコードの近くに差し込むのがコツ
- byebug gemを使ってシステムの状態を調査することは、アプリケーション内のエラーを追跡したりデバッグするときに非常に強力なツールになる

### マスアサインメント(7.3.1 解説) 
次のように値のハッシュを使ってRubyの変数を初期化するもの
マスアサインメントには**脆弱性**があるので**Rails 4.0 以降ではエラーになる**

マスアサインメントとは下記のようなコード

```@user = User.new(params[:user])```

下のコードとほぼ同じ

```@user = User.new(name: "Foo Bar", email: "foo@invalid", password: "foo", password_confirmation: "bar")```

Strong Parameters で対策することに

## Strong Parameters
必須のパラメータと許可されたパラメータを指定することができる
```Ruby
#:user必須
#名前、メールアドレス、パスワード、パスワードの確認の属性をそれぞれ許可
#それ以外は許可しない
params.require(:user).permit(:name, :email, :password, :password_confirmation)
```

## user_params
習慣として「user_params」という外部メソッドを使う params[:user]の代わりとして使われ、適切に初期化したハッシュを返してくれる Usersコントローラの内部でのみ実行されるためWeb経由で外部ユーザーにさらされない
 
```Ruby: app/controllers/users_controller.rb
class UsersController < ApplicationController
  .
  .
  .
  def create
    @user = User.new(user_params)
    if @user.save
      # 保存の成功をここで扱う。
    else
      render 'new'
    end
  end
 
  private
 
    def user_params
      params.require(:user).permit(:name, :email, :password,
                                   :password_confirmation)
    end
end
```

見つけやすくするためにprivateキーワード以降のインデントを一段下げている
（必須ではないけど見易くて良いっぽい）

## pluralize
```console: console
>> helper.pluralize(1, "error")
=> "1 error"
>> helper.pluralize(5, "error")
=> "5 errors"
```
pluralizeの最初の引数に整数が与えられると、それに基づいて2番目の引数の英単語を複数形に変更したものを返す
このメソッドの背後には強力なインフレクター (活用形生成) があり、不規則活用を含むさまざまな単語を複数形にすることができる

## リスト7.28 解説
```redirect_to @user``` = ```redirect_to user_url(@user)```

# flash 変数
``` flash[:success] = "Welcome to the Sample App!"```

ハッシュのように扱う変数
登録完了後に表示されるページにメッセージを表示し (この場合は新規ユーザーへのウェルカムメッセージ)、**2度目以降にはそのページにメッセージを表示しないようにする**というもの
Railsの一般的な慣習に倣って、:successというキーには成功時のメッセージを代入する (リスト 7.29)

### ERB 復習
```<% ... %>```: 中に書かれたコードを単に実行するだけで何も出力しない
```<%= ... %>```: 中のコードの実行結果がテンプレートのその部分に挿入される

ERBでビューをこのように書き換えても、ページの表示結果は以前とまったく同じ
タイトルの可変部分がERBによって動的に生成されている点だけが異なる

## assert_difference
```Ruby
assert_difference 'User.count', 1 do
  post users_path, ...
end
```
assert_no_differenceと同様に、このメソッドは第一引数に文字列 ('User.count') を取り、assert_differenceブロック内の処理を実行する直前と、実行した直後のUser.countの値を比較
第二引数はオプションですが、ここには比較した結果の差異 (今回の場合は1) を渡す

# Heroku にデプロイした後Application Error が出る
https://qiita.com/m-itoidcf/items/77d064147a32169b5449

解決策
一つ前のデプロイ状態に戻って、再度デプロイしなおした

```$ heroku rollback v12 (戻したいバージョン名)```

c.f. https://qiita.com/kimihito_/items/9608f2e94608c9359ad3
