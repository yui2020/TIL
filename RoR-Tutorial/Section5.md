# 10章 Summary
- ユーザーは、編集フォームからPATCHリクエストをupdateアクションに対して送信し、情報を更新する
- Strong Parametersを使うことで、安全にWeb上から更新させることができる
- beforeフィルターを使うと、特定のアクションが実行される直前にメソッドを呼び出すことができる
- beforeフィルターを使って、認可 (アクセス制御) を実現した
- 認可に対するテストでは、特定のHTTPリクエストを直接送信する低級なテストと、ブラウザの操作をシミュレーションする高級なテスト (統合テスト) の2つを利用した
- フレンドリーフォワーディングとは、ログイン成功時に元々行きたかったページに転送させる機能である
- ユーザー一覧ページでは、すべてのユーザーをページ毎に分割して表示する
- rails db:seedコマンドは、db/seeds.rbにあるサンプルデータをデータベースに流し込む
- render @usersを実行すると、自動的に_user.html.erbパーシャルを参照し、各ユーザーをコレクションとして表示する
- boolean型のadmin属性をUserモデルに追加すると、admin?という論理オブジェクトを返すメソッドが自動的に追加される
- 管理者が削除リンクをクリックすると、DELETEリクエストがdestroyアクションに向けて送信され、該当するユーザーが削除される
- fixtureファイル内で埋め込みRubyを使うと、多量のテストユーザーを作成することができる

## リスト10.2 コード解説
```shared/error_messages``` というパーシャルをrender (描画) 

Rails全般の慣習として、複数のビューで使われるパーシャルは専用のディレクトリ「shared」によく置かれます (実際このパーシャルは10.1.1でも使います)。

## target="_blank"
リンク先を新しいタブ (またはウィンドウ) で開くようになるので、別のWebサイトへリンクするときなどに便利です 

### target="_blank" のセキュリティ上の小さな問題
リンク先のサイトがHTMLドキュメントのwindowオブジェクトを扱えてしまう
c.f. フィッシング (Phising) サイト: 悪意のあるコンテンツを導入させられてしまう可能性があります。

対処方法: リンク用のaタグのrel (relationship) 属性に、"noopener"と設定するだけです。
e.g. ```<a href="http://gravatar.com/emails" rel="noopender" target="_blank">change</a>```

## new_record?
ユーザーが新規なのか、それともデータベースに存在する既存のユーザーであるかを、Active Recordのnew_record?論理値メソッドを使って区別できる

Railsは、form_for(@user)を使ってフォームを構成すると、@user.new_record?が
trueのときにはPOSTを、
falseのときにはPATCHを使います。

## リスト10.8 コード解説
```@user.update_attributes(user_params)```
update_attributesへの呼び出しでuser_paramsを使っている

### 復習 Strong Parameters
マスアサインメント脆弱性を防ぐ
https://railstutorial.jp/chapters/sign_up?version=5.1#sec-strong_parameters

## 受け入れテスト (Acceptance Tests)
「実装する前に」統合テストを書き、ある機能の実装が完了し、受け入れ可能な状態になったかどうかを決めるテスト

### 復習　@user.reload
データベースから最新のユーザー情報を読み込み直して、正しく更新されたかどうかを確認

## allow_nil オプション
パスワードのバリデーションに対して、空だったときの例外処理を加える。
[Rails バリデーションまとめ](https://qiita.com/ryuuuuuuuuuu/items/b7b985465fc0bf6dcaf8#allow_nil)

リスト 10.13によって、新規ユーザー登録時に空のパスワードが有効になってしまうのかと心配になるかもしれませんが、安心してください。
6.3.3で説明したように、has_secure_passwordでは (追加したバリデーションとは別に) オブジェクト生成時に存在性を検証するようになっているため、空のパスワード (nil) が新規ユーザー登録時に有効になることはありません。

# 10.2 認可
ウェブアプリケーションの文脈では、
- 認証 (authentication) はサイトのユーザーを識別することであり、
- 認可 (authorization) はそのユーザーが実行可能な操作を管理することです。

ユーザーにログインを要求し、自分以外のユーザー情報を変更できないように制御する
→セキュリティ上の制御機構をセキュリティモデルと呼ぶ

## before フィルターとは
before_actionメソッドを使って何らかの処理が実行される直前に特定のメソッドを実行する仕組みです
デフォルトでは、beforeフィルターはコントローラ内のすべてのアクションに適用される

e.g.
```rb
class UsersController < 
  # 直前にlogged_in_userメソッドを実行　onlyオプションでedit, updateにのみ適用
  before_action :logged_in_user, only:[:edit, :update]
.
.
.
    
# beforeアクション
 
    # ログイン済みユーザーかどうか確認
    def logged_in_user
      # logged_in?メソッドがfalseの場合
      unless logged_in?
        # flashｓでエラーメッセージを表示
        flash[:danger] = "Please log in."
        # login_urlにリダイレクト
        redirect_to login_url
      end
    end

```
デフォルトだとコントローラー内全てのアクションに適用されるので
ここでは適切な:onlyオプション (ハッシュ) を渡すことで、:editと:updateアクションだけにこのフィルタが適用されるように制限をかけています。

## unless文
if文が条件式の評価がtrueの場合に処理を実行するのに対して unless文は**条件式の評価がfalseの場合に処理を実行する** 
上のコードの場合は、 ヘルパーに定義したlogged_in?メソッドがfalse（current_userがnil）の場合に処理（エラーメッセージの表示→login_urlにリダイレクト）が行われる

# フレンドリーフォワーディング (friendly forwarding)
各種リダイレクト先はユーザーが開こうとしていたページに設定するのが親切
ログインしていないユーザーが編集ページにアクセスしようとしていたなら、ユーザーがログインした後にはその編集ページにリダイレクトされるようにするのが望ましい

## フレンドリーフォワーディングの実装
ユーザーを希望のページに転送するには、リクエスト時点のページをどこかに保存しておき、その場所にリダイレクトさせる必要があります。
store_locationとredirect_back_orの2つのメソッドを使います（Sessionsヘルパーで定義しています (リスト 10.30)）

## request オブジェクト
- request.original_url:　現在のリクエストURLをStringとして返す
- request.get?:　HTTPメソッドがGETの時trueを返す

## redirect_back_orメソッド
フォワーディング自体を実装するには、redirect_back_orメソッドを使います。
リクエストされたURLが存在する場合はそこにリダイレクトし、
ない場合は何らかのデフォルトのURLにリダイレクトします。
デフォルトのURLは、Sessionコントローラのcreateアクションに追加し、サインイン成功後にリダイレクトします (リスト 10.32)。redirect_back_orメソッドでは、次のようにor演算子||を使います。
```session[:forwarding_url] || default```

このコードは、値がnilでなければsession[:forwarding_url]を評価し、そうでなければデフォルトのURLを使っています

# すべてのユーザーを表示する
## ページネーション (pagination=ページ分割)
将来ユーザー数が膨大になってもindexページを問題なく表示できるようにするためのユーザー出力

 ## Faker gem
実際にいそうな名前のユーザー名を作成するgem
faker gemは開発環境以外では普通使いませんが、今回は例外的に本番環境でも適用させる予定 (10.5) なので、次のようにすべての環境で使えるようにしています。

### create! メソッド
create!は基本的にcreateメソッドと同じものですが、ユーザーが無効な場合にfalseを返すのではなく例外を発生させる (6.1.4) 点が異なります。
こうしておくと見過ごしやすいエラーを回避できるので、デバッグが容易になります。

# ページネーション
例えば1つのページに一度に30人だけユーザーを表示するというものです。
1つのページに膨大な数のユーザーが表示されるのを防ぐ目的

## ページネーションの実装
ユーザーのページネーションを行うようにRailsに指示するコードをindexビューに追加する必要があります。
また、indexアクションにあるUser.allを、ページネーションを理解できるオブジェクトに置き換える必要もあります。

```rb: app/views/users/index.html.erb
<% provide(:title, 'All users') %>
<h1>All users</h1>

<%= will_paginate %>

<ul class="users">
  <% @users.each do |user| %>
    <li>
      <%= gravatar_for user, size: 50 %>
      <%= link_to user.name, user %>
    </li>
  <% end %>
</ul>

<%= will_paginate %>
```

```
$ rails console
>> User.paginate(page: 1)
  User Load (1.5ms)  SELECT "users".* FROM "users" LIMIT 30 OFFSET 0
   (1.7ms)  SELECT COUNT(*) FROM "users"
=> #<ActiveRecord::Relation [#<User id: 1,...
```
paginateでは、キーが:pageで値がページ番号のハッシュを引数に取ります。
User.paginateは、:pageパラメーターに基いて、データベースからひとかたまりのデータ (デフォルトでは30) を取り出します。
したがって、1ページ目は1から30のユーザー、2ページ目は31から60のユーザーといった具合にデータが取り出されます。
ちなみにpageがnilの場合、 paginateは単に最初のページを返します。

## アクションの編集
will_paginateメソッドはusersビューのコードの中から@usersオブジェクトを自動的に見つけ出してページネーションリンクを作成してくれる。すごい！ ただし、will_paginateではpaginateメソッドを使った結果が必要な為@usersに代入してあげる必要がある

## リスト 10.50: indexビューに対する最初のリファクタリング
```rb: app/views/users/index.html.erb
<% provide(:title, 'All users') %>
<h1>All users</h1>

<%= will_paginate %>

<ul class="users">
  <% @users.each do |user| %>
    <%= render user %>
  <% end %>
</ul>

<%= will_paginate %>
```

ここでは、renderをパーシャル (ファイル名の文字列) に対してではなく、Userクラスのuser変数に対して実行している点に注目してください12 。
この場合、Railsは自動的に_user.html.erbという名前のパーシャルを探しにいくので、このパーシャルを作成する必要があります (リスト 10.51)。

# 10.4.1 管理ユーザー
論理値をとるadmin属性をUserモデルに追加する　→　自動的に論理値を返す
admin?メソッドも使えるようになる　→　管理ユーザーの状態をテストできる

## Strong Parameters
もし、任意のWebリクエストの初期化ハッシュをオブジェクトに渡せるとなると、攻撃者は次のようなPATCHリクエストを送信してくるかもしれません13 。
```patch /users/17?admin=1```
このリクエストは、17番目のユーザーを管理者に変えてしまいます。
このような危険があるからこそ、Strong Parameter を使って編集してもよい安全な属性だけを更新することが重要になります。

```rb
    def user_params
      params.require(:user).permit(:name, :email, :password,
                                   :password_confirmation)
    end
```
paramsハッシュに対してrequireとpermitを呼び出します。

上のコードでは、許可された属性リストにadminが含まれていない
これにより、任意のユーザーが自分自身にアプリケーションの管理者権限を与えることを防止できます。
この問題は重大であるため、編集可能になってはならない属性に対するテストを作成することをぜひともオススメします。

# destroyアクション
ブラウザはネイティブではDELETEリクエストを送信できないため、RailsではJavaScriptを使って偽造します。
= JavaScriptがオフになっているとユーザー削除のリンクも無効になるということです。

JavaScriptをサポートしないブラウザをサポートする必要がある場合は、フォームとPOSTリクエストを使ってDELETEリクエストを偽造することもできます。こちらはJavaScriptがなくても動作します14 。
c.f. http://railscasts.com/episodes/77-destroy-without-javascript

## assert_no_difference メソッド
ユーザー数が変化しないことを確認

```rb
# ブロックで渡されたものを呼び出す前後でUser.countに違いがない
    assert_no_difference 'User.count' do
```

## assert_difference
ユーザー数が変化することを確認

```rb
assert_difference 'User.count', -1 do
  delete user_path(@other_user)
end
```
管理者が削除リンクをクリックしたときに、ユーザーが削除されたことを確認する
DELETEリクエストを適切なURLに向けて発行し、User.countを使ってユーザー数が 1つ 減ったかどうかを確認

## この時点でテストに失敗
エラーメッセージ
```console
ERROR["test_should_redirect_destroy_when_not_logged_in", UsersControllerTest, 1.188652970000021]
 test_should_redirect_destroy_when_not_logged_in#UsersControllerTest (1.19s)
NoMethodError:         NoMethodError: undefined method `admin?' for nil:NilClass
            app/controllers/users_controller.rb:80:in `admin_user'
            test/controllers/users_controller_test.rb:49:in `block (2 levels) in <class:UsersControllerTest>'
            test/controllers/users_controller_test.rb:48:in `block in <class:UsersControllerTest>'
```
解決策: [Ruby on Rails チュートリアル第10章　10.4.3ユーザー削除テストでエラーが出てしまいます](https://teratail.com/questions/177179)

```before_action :logged_in_user, only: [:index, :edit, :update, :destroy]```
:destroy を追加



