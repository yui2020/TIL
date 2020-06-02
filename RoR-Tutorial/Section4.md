# 9章 Summary
- Railsでは、あるページから別のページに移動するときに状態を保持することができる。ページの状態を長期間保持したいときは、cookiesメソッドを使って永続的なセッションにする
- 記憶トークンと記憶ダイジェストをユーザーごとに関連付けて、永続的セッションが実現できる
- cookiesメソッドを使うと、ユーザーのブラウザにcookiesなどを保存できる
- ログイン状態は、セッションもしくはクッキーの状態に基づいて決定される
- セッションとクッキーをそれぞれ削除すると、ユーザーのログアウトが実現できる
- 三項演算子を使用すると、単純なif-then文をコンパクトに記述できる

# remember me
(任意で) ユーザーのログイン情報を記憶しておき、ブラウザを再起動した後でもすぐにログインできる機能 
永続クッキー (permanent cookies) を使ってこの機能を実現

## セッションの永続化
記憶トークン (remember token) を生成し、cookiesメソッドによる永続的cookiesの作成や、安全性の高い記憶ダイジェスト (remember digest) によるトークン認証にこの記憶トークンを活用します

sessionメソッドで保存した情報は自動的に安全が保たれますが、cookiesメソッドに保存する情報は残念ながらそのようにはなっていません。

特に、cookiesを永続化するとセッションハイジャックという攻撃を受ける可能性があります。

# セッションハイジャック
記憶トークンを奪って、特定のユーザーになりすましてログインするというものです。

cookiesを盗み出す有名な方法は4通りあります。 
1. 管理の甘いネットワークを通過するネットワークパケットからパケットスニッファという特殊なソフトウェアで直接cookieを取り出す1 
2. データベースから記憶トークンを取り出す。
3. クロスサイトスクリプティング (XSS) を使う。 
4. ユーザーがログインしているパソコンやスマホを直接操作してアクセスを奪い取る。

- Secure Sockets Layer (SSL) をサイト全体に適用して、ネットワークデータを暗号化で保護し、パケットスニッファから読み取られないようにしています
- 記憶トークンをそのままデータベースに保存するのではなく、記憶トークンのハッシュ値を保存するようにします。これは、6.3で生のパスワードをデータベースに保存する代わりに、パスワードのダイジェストを保存したのと同じ考え方に基づいています2 。
- Railsによって自動的に対策が行われます。具体的には、ビューのテンプレートで入力した内容をすべて自動的にエスケープします。
- ログイン中のコンピュータへの物理アクセスによる攻撃については、さすがにシステム側での根本的な防衛手段を講じることは不可能なのですが、二次被害を最小限に留めることは可能です。具体的には、ユーザーが (別端末などで) ログアウトしたときにトークンを必ず変更するようにし、セキュリティ上重要になる可能性のある情報を表示するときはデジタル署名 (digital signature) を行うようにします。

## 永続的セッション作成方針
1. 記憶トークンにはランダムな文字列を生成して用いる。
2. ブラウザのcookiesにトークンを保存するときには、有効期限を設定する。
3. トークンはハッシュ値に変換してからデータベースに保存する。
4. ブラウザのcookiesに保存するユーザーIDは暗号化しておく。
5. 永続ユーザーIDを含むcookiesを受け取ったら、そのIDでデータベースを検索し、記憶トークンのcookiesがデータベース内のハッシュ値と一致することを確認する（has_secure_passwordと似てる！）

### マイグレーションを生成 復習
```$ rails generate migration add_remember_digest_to_users remember_digest:string```

前回と同様、今回のマイグレーション名も_to_usersで終わっています。
これは、マイグレーションの対象が**データベースのusersテーブルであることをRailsに指示**するためのものです。
今回は種類=stringのremember_digest属性を追加しているので、いつものようにRailsによってデフォルトのマイグレーションが作成されます (リスト 9.1)。

記憶ダイジェストはユーザーが直接読み出すことはないので (かつ、そうさせてはならないので)、**remember_digestカラムにインデックスを追加する必要はありません。**

## urlsafe_base64
Ruby標準ライブラリのSecureRandomモジュールにあるurlsafe_base64メソッドは、A–Z、a–z、0–9、"-"、"_"のいずれかの文字 (64種類) からなる長さ22のランダムな文字列を返します (64種類なのでbase64と呼ばれています)。
```
$ rails console
>> SecureRandom.urlsafe_base64
=> "q5lt38hQDc_959PVoo6b7A"
```
今回は記憶トークンに利用

## attr_accessor 復習
「仮想の」属性を作成
attr_accessor解説
- [RoRT本文](https://railstutorial.jp/chapters/rails_flavored_ruby?version=5.1#sec-a_user_class)
- [らくだ🐫にもできるRailsチュートリアル｜4.4（と4.5）](https://rakuda3desu.net/rakudas-rails-tutorial4-4/)

## コード解説
```rb
class User < ApplicationRecord
  #remember_token属性をUserクラスに定義
  attr_accessor :remember_token
  .
  .
  .
  def remember
    #remember_tokenに要素を代入
    #（selfを付けるとクラス変数になる→この場合User.remember_tokenと同意）
    self.remember_token = ...
    #validationを無視して更新（引数にハッシュで更新したい属性と値をセット）
    update_attribute(:remember_digest, ...)
  end
end
```
- selfというキーワードを使わないと、Rubyによってremember_tokenという名前のローカル変数が作成されてしまいます。 ( = Rubyにおけるオブジェクト内部への要素代入)
- update_attributeメソッドを使って記憶ダイジェストを更新しています。6.1.5で説明したように、**update_attributeメソッドはバリデーションを素通りさせます。**今回はユーザーのパスワードやパスワード確認にアクセスできないので、バリデーションを素通りさせなければなりません。

### 省略されるselfと省略できないself

```rb
def remember
    #remember_tokenに　User.new_tokenを代入
    self.remember_token = User.new_token
    #validationを無視して更新　（:remember_digest属性にハッシュ化したremember_tokenを）
    update_attribute(:remember_digest, User.digest(remember_token))
  end
```
selfの有無によって　self.remember_token（インスタンス変数）　remember_token（メソッド内の変数）になっちゃうから 30行目の(remember_token)にself.つけないとじゃない？っていう話が出まして **本来はselfついてるけど省略**されているよ！ということのようです。 
c.f. [rubyでselfを省略できる時、できない時](https://qiita.com/akira-hamada/items/4132d2fda7e420073ab7)

[リスト9.4  コード](https://railstutorial.jp/chapters/advanced_login?version=5.1#code-token_digest_self)
```rb
class User < ApplicationRecord
  .
  .
  .
  # 渡された文字列のハッシュ値を返す
  #User.digest(string)と同じ意味
  # ここでのself = Userクラス
  def self.digest(string)
    cost = ActiveModel::SecurePassword.min_cost ? BCrypt::Engine::MIN_COST :
                                                  BCrypt::Engine.cost
    BCrypt::Password.create(string, cost: cost)
  end
 
  # ランダムなトークンを返す
  #User.new_tokenと同じ意味
  def self.new_token
    SecureRandom.urlsafe_base64
  end
  .
  .
  .
end
```

[リスト9.5 コード](https://railstutorial.jp/chapters/advanced_login?version=5.1#code-token_digest_class_self)
```rb
class User < ApplicationRecord
  .
  .
  .
  #このスコープの中に書かれたメソッドはクラスメソッドになる
  class << self
    # 渡された文字列のハッシュ値を返す
    def digest(string)
      cost = ActiveModel::SecurePassword.min_cost ? BCrypt::Engine::MIN_COST :
                                                    BCrypt::Engine.cost
      BCrypt::Password.create(string, cost: cost)
    end
 
    # ランダムなトークンを返す
    def new_token
      SecureRandom.urlsafe_base64
    end
  end
```

## cookiesメソッド
永続セッションを作成する
sessionのときと同様にハッシュとして扱えます。
個別のcookiesは、１つのvalue (値) と、オプションのexpires (有効期限) からできています。有効期限は省略可能です。

e.g. 20年後に期限切れになる記憶トークンと同じ値をcookieに保存することで、永続的なセッションを作る

```rb
#20年後に期限切れになる設定
cookies[:remember_token] = { value:   remember_token,
                             expires: 20.years.from_now.utc }
 
#よく使われる設定なのでpermanentという専用メソッドが追加されている
#20年後に期限切れになる設定のコードを更にコンパクトに
cookies.permanent[:remember_token] = remember_token
```

c.f. [たのしいOSSコードリーディング: Let's read "cookies"🍪
](https://qiita.com/coe401_/items/ad7dc2f3e319c5beaf40)

# 署名付きcookie
cookieをブラウザに保存する前に安全に暗号化するためのものです8 。
b/c IDが生のテキストとしてcookiesに保存されてしまうのを防ぐため

```rb
#ユーザーIDをcookiesに保存
cookies[:user_id] = user.id
 
#↑このままではIDが生のテキストとしてcookiesに保存されるため「署名付きcookie」を使い暗号化する
#signedはデジタル署名と暗号化をまとめて実行してくれる
cookies.signed[:user_id] = user.id
 
#ユーザーIDと記憶トークンはペアで扱うのでcookieも永続化する必要がある
#よってsignedとpermanentをメソッドチェーンで繋いで使う
cookies.permanent.signed[:user_id] = user.id
 
#cookiesを設定しておく事で、cookiesからuserを取り出すことが出来るようになる
#自動的にユーザーIDのcookiesの暗号が解除され元に戻る
User.find_by(id: cookies.signed[:user_id])
```

# リスト9.6 コード解説
```rb
    # 渡されたトークンがダイジェストと一致したらtrueを返す
    def authenticated?(remember_token)
      # is_password?の引数remember_token はメソッド内のローカル変数を参照
      # remember_digest = self.remember_digest
      BCrypt::Password.new(remember_digest).is_password?(remember_token)
    end
```
# リスト9.8 解説
永続セッションの場合は、session[:user_id]が存在すれば一時セッションからユーザーを取り出し、
それ以外の場合はcookies[:user_id]からユーザーを取り出して、対応する永続セッションにログインする必要があります

```@current_user ||= User.find_by(id: session[:user_id])```

```rb
# 比較 (==) ではなく
# (ユーザーIDにユーザーIDのセッションを代入した結果) ユーザーIDのセッションが存在すれば」という意味
if (user_id = session[:user_id])
  @current_user ||= User.find_by(id: user_id)
elsif (user_id = cookies.signed[:user_id])
  #userに　cookiesから取り出したidの値がUser.idと一致するユーザーを代入
  user = User.find_by(id: user_id)
  if user && user.authenticated?(cookies[:remember_token])
    log_in user
    @current_user = user
  end
end
```

# FATAL: Listen error の解決法
FATAL: Listen error が発生し、rails sever が起動できなくなった（consoleは起動できる）

[rails consoleが起動できない時の対処法（「FATAL: Listen error」エラー編）](https://qiita.com/yn-misaki/items/c850a07f7858437e4d26)

## ユーザーの取り消し　user.forget
user.forgetメソッドによって、user.rememberが取り消されます。
具体的には、記憶ダイジェストをnilで更新します

## assert_equal
assert_equalの引数は「期待する値, 実際の値」の順序で書く点に注意してください。
```assert_equal <expected>, <actual>```

## この時点でtestがパスしない！！
原因: ```include SessionHelper``` がなかった
c.f. https://teratail.com/questions/131553

これ見落としてて3h くらいぐるぐるしてた、、

# Heroku メンテナンスモード
```heroku maintenance:on``` : オン
```heroku maintenance:off``` : オフ
Herokuにデプロイしても、Heroku上でマイグレーションを実行するまでの間は一時的にアクセスできない状態 (エラーページ) になる
トラフィックの多い本番サイトでは、このような変更を行う前にメンテナンスモードをオンにしておくことが一般的

