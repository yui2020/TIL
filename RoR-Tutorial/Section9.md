# 14章 Summary
- has_many :throughを使うと、複雑なデータ関係をモデリングできる
- has_manyメソッドには、クラス名や外部キーなど、いくつものオプションを渡すことができる
- 適切なクラス名と外部キーと一緒にhas_many/has_many :throughを使うことで、能動的関係 (フォローする) や受動的関係 (フォローされる) がモデリングできた
- ルーティングは、ネストさせて使うことができる
- whereメソッドを使うと、柔軟で強力なデータベースへの問い合わせが作成できる
- Railsは (必要に応じて) 低級なSQLクエリを呼び出すことができる
- 本書で学んだすべてを駆使することで、フォローしているユーザーのマイクロポスト一覧をステータスフィードに表示させることができた

# 14.1.4 フォローしているユーザー
図 14.7のように、1人のユーザーにはいくつもの「フォローする」「フォローされる」といった関係性があります (こういった関係性を「多対多」と呼びます)。
デフォルトのhas_many throughという関連付けでは、Railsはモデル名 (単数形) に対応する外部キーを探します

## has_many through
この関連付けは、2つのモデルの間に「第3のモデル」(joinモデル)が介在する点が特徴
それによって、相手モデルの「0個以上」のインスタンスとマッチします。
has_many :through関連付けは、他方のモデルと「多対多」のつながりを設定する場合によく使われる

cf. 
- [has_many :through 関連付け](https://railsguides.jp/association_basics.html#has-many-through%E9%96%A2%E9%80%A3%E4%BB%98%E3%81%91)
- [Rails4のhas_many throughで多対多のリレーションを実装する](https://qiita.com/samurai_runner/items/cbd91bb9e3f8b0433b99)

***

```has_many :followeds, through: :active_relationships```

Railsはモデル名 (単数形) に対応する外部キーを探します。
つまり、Railsは「followeds」というシンボル名を見て、これを「followed」という単数形に変え、 relationshipsテーブルのfollowed_idを使って対象のユーザーを取得してきます。


### user.following 
しかし、14.1.1で指摘したように、user.followedsという名前は英語として不適切です。
代わりに、user.followingという名前を使いましょう。


has_many :throughの関連付けにより、フォローしているユーザーを配列の様に扱えるようになりました。

```
# include: フォローしているユーザーの集合を調べたり、関連付けを通してオブジェクトを探せる
user.following.include?(other_user)
user.following.find(other_user)

# followingで取得したオブジェクトは、配列のように要素の追加や削除ができる
user.following << other_user
user.following.delete(other_user)

# フォローしている全てのユーザーをデータベースから取得し、その集合に対してinclude?メソッドを実行（実際はdbの中で直接比較）
following.include?(other_user)
```


# リスト14.15 解説
```rb:config/routes.rb
  resources :users do
    # ユーザーidが含まれるurlを扱う
    member do
      get :following, :followers
    end
  end
```

```rb
 resources :users do
# idを指定せずに全てのメンバーを表示する
  collection do
    get :tigers
  end
end
```
このコードは /users/tigers というURLに応答します (アプリケーションにあるすべてのtigerのリストを表示します)7 。

# リスト14.21 コード解説
```rb:app/views/users/_follow.html.erb
<%= form_for(current_user.active_relationships.build) do |f| %>
  <div><%= hidden_field_tag :followed_id, @user.id %></div>
  <%= f.submit "Follow", class: "btn btn-primary" %>
<% end %>
```

## hidden_field_tag メソッド
```rb
<div><%= hidden_field_tag :followed_id, @user.id %></div>
```

上のコードは以下のhtmlを生成

```html
# inputタグ: 隠しフィールド（ブラウザ上に表示させず、適切な情報を含めることができる）
<input id="followed_id" name="followed_id" type="hidden" value="3" />
```

# 14.2.5 [Follow] ボタン (Ajax編)

## Ajax とは
Ajaxを使えば、Webページからサーバーに「非同期」で、ページを移動することなくリクエストを送信することができます

```rb
form_for

# Ajaxを使うにはremote: true にするだけ！
form_for …, remote: true
```

## respond_to メソッド
リクエストの種類によって応答を場合分けできる

```rb
respond_to do |format|
  format.html { redirect_to user }
  format.js
end
```

上の (ブロック内の) コードのうち、**いずれかの1行が実行される**という点が重要です 
respond_toメソッドは、上から順に実行する逐次処理というより、if文を使った分岐処理に近いイメージ！


Ajaxリクエストを受信した場合は、Railsが自動的にアクションと同じ名前を持つJavaScript用の埋め込みRuby (.js.erb) ファイル (create.js.erbやdestroy.js.erbなど) を呼び出す

```rb
# $ = jQuery, id = follow_form にアクセス
$(“#follow_form”)
```
これはフォームを囲むdivタグであり、フォームそのものではない！

### htmlメソッド
引数の中で指定された要素の内側にあるHTMLを更新します

```rb
$("#follow_form").html("foobar")
```

# リスト 14.38: JavaScriptと埋め込みRubyを使ってフォローの関係性を作成する

```rb:app/views/relationships/create.js.erb

$("#follow_form").html("<%= escape_javascript(render('users/unfollow')) %>");
$("#followers").html('<%= @user.followers.count %>');
```
各行の末尾にセミコロン ; があることに注意！

### escape_javascriptメソッド
JavaScriptファイル内にHTMLを挿入するときに実行結果をエスケープする


# 14.2.6 フォローをテストする

Ajax 版のテスト: **xhr: true** オプションを使う

```rb
assert_difference '@user.following.count', 1 do
  post relationships_path, params: { followed_id: @other.id }, xhr: true
end
```

xhr (XmlHttpRequest) というオプションをtrueに設定すると、Ajaxでリクエストを発行するように変わります。


# 14.3.2 フィードを初めて実装する
```SELECT * FROM microposts WHERE user_id IN (<list of ids>) OR user_id = <user id>```

micropostsテーブルから、あるユーザー (つまり自分自身) がフォローしているユーザーに対応するidを持つマイクロポストをすべて選択 (select) する


## following_idsメソッド
has_many :followingの関連付けをしたときにActive Recordが自動生成したもの

user.followingコレクションに対応するidを得るためには、関連付けの名前の末尾に_idsを付け足すだけで済みます


# 14.3.3 サブセレクト
following_idsでフォローしているすべてのユーザーをデータベースに問い合わせし、さらに、フォローしているユーザーの完全な配列を作るために再度データベースに問い合わせしているという問題点が発生する
もっと効率的なコードに置き換えられるはず！！


## SQLのサブセレクト (subselect) 

```rb
# これまでのコード
Micropost.where("user_id IN (?) OR user_id = ?", following_ids, id)

# 置き換えたコード
Micropost.where("user_id IN (:following_ids) OR user_id = :user_id",
    following_ids: following_ids, user_id: id)
```

```rb
# following_ids は 次のコードに置き換えられる
following_ids = "SELECT followed_id FROM relationships
                 WHERE  follower_id = :user_id"
```
このコードをSQLのサブセレクトとして使う

つまり、「ユーザー1がフォローしているユーザーすべてを選択する」というSQLを既存のSQLに内包させる

結果、SQLは次のようになる
```sql
SELECT * FROM microposts
WHERE user_id IN (SELECT followed_id FROM relationships
                  WHERE  follower_id = 1)
                  OR user_id = 1
```
