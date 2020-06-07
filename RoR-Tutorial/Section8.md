# 13章 Summary
- Active Recordモデルの力によって、マイクロポストも (ユーザーと同じで) リソースとして扱える
- Railsは複数のキーインデックスをサポートしている
- Userは複数のMicropostsを持っていて (has_many)、Micropostは1人のUserに依存している (belongs_to) といった関係性をモデル化した
- has_manyやbelongs_toを利用することで、関連付けを通して多くのメソッドが使えるようになった
- user.microposts.build(...)というコードは、引数で与えたユーザーに関連付けされたマイクロポストを返す
- default_scopeを使うとデフォルトの順序を変更できる
- default_scopeは引数に無名関数 (->) を取る
- dependent: :destroyオプションを使うと、関連付けされたオブジェクトと自分自身を同時に削除する
- paginateメソッドやcountメソッドは、どちらも関連付けを通して実行され、効率的にデータベースに問い合わせしている
- fixtureは、関連付けを使ったオブジェクトの作成もサポートしている
- パーシャルを呼び出すときに、一緒に変数を渡すことができる
- whereメソッドを使うと、Active Recordを通して選択 (部分集合を取り出すこと) ができる
- 依存しているオブジェクトを作成/削除するときは、常に関連付けを通すようにすることで、よりセキュアな操作が実現できる
- CarrierWaveを使うと画像アップロードや画像リサイズができる


# Bundler::GemNotFoundが出てrails  g できない fixed!

状況:
error message
Could not find sassc-2.4.0 in any of the sources (Bundler::GemNotFound)

原因:
Github  のbot による自動プルリクエストをマージしてたから
	プルリク内容: Bump bootstrap-sass from 3.3.7 to 3.4.1 #3

対策:
```
$ bundle update
$ bundle install

$ rails g hoge hoge 
```
で解決しました！

cf. [dockerでBundler::GemNotFoundが出るときの対処法](https://qiita.com/koyo-miyamura/items/5f1d123046917782e111)


# モデルの生成
```$ rails generate model Micropost content:text user:references```

references型
これを利用すると、自動的にインデックスと外部キー参照付きのuser_idカラムが追加され3 、UserとMicropostを関連付けする下準備をしてくれます。

```rb
class CreateMicroposts < ActiveRecord::Migration[5.1]
  def change
    create_table :microposts do |t|
      t.text :content
      t.references :user, foreign_key: true
 
      t.timestamps
    end
    # user_idとcreated_atカラムにインデックスを付与（この↓一行追加)
    add_index :microposts, [:user_id, :created_at]
  end
end
```

インデックスを付与することでuser_idに関連付けられたすべてのマイクロポストを作成時刻の逆順で取り出しやすくしている

Active Recordは、両方のキーを同時に扱う複合キーインデックス (Multiple Key Index) を作成


## Sanity check
正常な状態かどうかをテスト (sanity check) 

# User/Micropost の関連付け

￼
図 13.2: MicropostとそのUserは belongs_to (1対1) の関係性がある
￼
図 13.3: UserとそのMicropostは has_many (1対多) の関係性がある


## belongs_to, has_many
紐付いたユーザーを通してマイクロポストを作成することが出来るようになる
Micropostモデル単体ではこうだったメソッドが
```
Micropost.create
Micropost.create!
Micropost.new
```

こうなる
```
user.microposts.create
user.microposts.create!
user.microposts.build
```

新規のマイクロポストが上記のメソッドで作成されると
user_idに**自動的**に**正しい値**が設定される

この方法を使うと
```rb
@user = users(:michael)
# このコードは慣習的に正しくない
@micropost = Micropost.new(content: "Lorem ipsum", user_id: @user.id)
```

という書き方 (リスト 13.4) が、

```rb
@user = users(:michael)
@micropost = @user.microposts.build(content: "Lorem ipsum")
```

 ### buildメソッド
オブジェクトを返しますがデータベースには反映されません。

Micropostモデルに必要なbelongs_to :userというコードは自動的に生成されている
Userモデルに必要なhas_many :micropostsは手動で追加する


## デフォルトのスコープ
user.micropostsメソッドはデフォルトでは呼び出しの順序に何の保証もない（決まっていない） →一般的な表示順として新しい順に表示されるようにする これを実装するためにdefault scopeというテクニックを使う！

### default_scopeメソッド
データベースから要素を取得したときの、デフォルトの順序を指定するメソッド

特定の順序にしたい場合は、default_scopeの引数にorderを与えます。

```order(:created_at)```
created_atカラムの順にする
デフォルトの順序が昇順 (ascending) = 数の小さい値から大きい値にソート

```order(created_at:  :desc)```
desc: SQLの降順 (descending) = 数の小さい値から大きい順にソート


# ラムダ式 (Stabby lambda) とは何ぞ
Procやlambda (もしくは無名関数)と呼ばれるオブジェクトを作成する文法の事

### Procオブジェクトとは
Procオブジェクトとはブロックをオブジェクト化したProcクラスのインスタンス

->というラムダ式は、ブロック (4.3.2) を引数に取り、Procオブジェクトを返します。
このオブジェクトは、callメソッドが呼ばれたとき、ブロック内の処理を評価します。

```
# 実際は default_scope -> { hogehoge } な形
>> -> { puts "foo" }
=> #<Proc:0x007fab938d0108@(irb):1 (lambda)>
>> -> { puts "foo" }.call
foo
=> nil
```

cf. [[Ruby]Procオブジェクトについて](https://qiita.com/nagata03/items/919b0457409cc755ff0f)


## dependent: :destroyオプション
削除されたものに紐づいたものも同様に削除される
ユーザーが削除されたときに、そのユーザーに紐付いた (そのユーザーが投稿した) マイクロポストも一緒に削除されるようになります

### micropostを削除させないように指定することもできる！
cf. [dependentオプションまとめ](https://qiita.com/kobukurosirokuma/items/b86ac6b620d2b15c0164)

# 

```rb
# ユーザー一覧ページで使ったページネーションのコード
<%= will_paginate %>
 
# 今回使うページネーションのコード
<%= will_paginate @microposts %>
```
ユーザー一覧ページのコードは引数なしで動作
b/c will_paginateが、Usersコントローラのコンテキストにおいて、@usersインスタンス変数が存在していることを前提としているため

ActiveRecord::Relationクラスの@usersインスタンス変数はUsersコントローラのコンテキスト（Usersコントローラに含まれる情報）なので、
同じくUsersコントローラのコンテキスト（であるところのViewのテンプレート上）からwill_paginateを呼び出しているため（@usersが存在するという前提で）will_paginateがよしなにやってくれていた
 今回はUsersコントローラのコンテキストから、異なるコンテキストのマイクロポストをページネーションしたいので （明示的に）@microposts変数をwill_paginateに渡す必要がある


### take()
要素を配列にして先頭から引数の数だけ返す

```
# 1-10を　配列にして　先頭から6個を取り出す
>> (1..10).to_a.take(6)
=> [1, 2, 3, 4, 5, 6]

# takeが「要素を配列にして先頭から引数個返す」ためto_aは不要
>> (1..10).take(6)
=> [1, 2, 3, 4, 5, 6]
```


# エラーメッセージのパーシャルを定義

_micropost_form.html.erb内のこのコード↓が動くようにするため

```<%= render 'shared/error_messages', object: f.object %>```

現在のエラーパーシャルはこう

```rb
<% if @user.errors.any? %>
  <div id="error_explanation">
    <div class="alert alert-danger">
       <%= t('.errors count', errors_count: @user.errors.count) %>
    </div>
    <ul>
    <% @user.errors.full_messages.each do |msg| %>
      <li><%= msg %></li>
    <% end %>
    </ul>
  </div>
<% end %>
```

このままではviewのこの↓コードが動かない

```<%= render 'shared/error_messages', object: f.object %>```

元々のエラーメッセージパーシャルでは@users変数を直接参照している
→今回は@micropost変数を参照したい！
→@users変数でも@micropostでも参照できたい！
→フォーム変数fをf.objectにして関連付けられたオブジェクトにアクセスできるようにする！

```rb
<% if object.errors.any? %>
  <div id="error_explanation">
    <div class="alert alert-danger">
      The form contains <%= pluralize(object.errors.count, "error") %>.
    </div>
    <ul>
    <% object.errors.full_messages.each do |msg| %>
      <li><%= msg %></li>
    <% end %>
    </ul>
  </div>
<% end %>
```

error_messagesパーシャルを更新してもテストはRED
→error_messagesパーシャルは他のviewからも呼び出されているため

```rb:app/views/users/_form.html.erb
<!--yieldメソッドで個別に設定したurlを渡す-->
<%= form_for(@user, url: yield(:url)) do |f| %>
    <%= render 'shared/error_messages', object: f.object %>

  <%= f.label :name %>
  <%= f.text_field :name, class: 'form-control' %>
```

```rb:app/views/password_resets/edit.html.rb
<% provide(:title, t('.reset_password_title')) %>
<h1><%= t('.reset_password_title') %></h1>

<div class="row">
  <div class="col-md-6 col-md-offset-3">
    <%= form_for(@user, url: password_reset_path(params[:id])) do |f| %>
      <%= render 'shared/error_messages', object: f.object %>

      <%= hidden_field_tag :email, @user.email %>
・
・
・
```

# 13.3.3 フィードの原型

```rb
Micropost.where("user_id = ?", id)
```
上のコードの記述はセキュリティ上とっても重要！

SQLクエリに代入する前にidがエスケープされ
SQLインジェクション (SQL Injection) と呼ばれる深刻なセキュリティホールを避けることができます。

この場合のid属性は単なる整数 (すなわちself.idはユーザーのid) であるため危険はありませんが、**SQL文に変数を代入する場合は常にエスケープする**習慣をぜひ身につけてください。

## SQLインジェクションとは何ぞ
穴埋めになっているSQL文の穴埋め部分に、作った人が意図しない内容を入れることによって、おかしな動きをさせること。
もしくは、それができるようになっている状態


## Micropost.where(“user_id = ?”, id)とはなんぞ
プレースホルダー（?　の事）を使った書き方 第一引数の user_id =　？　の？に、第2引数のid が置き換わる

cf. [whereメソッドを徹底解説！ -プレースホルダの記述の仕方](https://pikawaka.com/rails/where#%E3%83%97%E3%83%AC%E3%83%BC%E3%82%B9%E3%83%9B%E3%83%AB%E3%83%80%E3%81%AE%E8%A8%98%E8%BF%B0%E3%81%AE%E4%BB%95%E6%96%B9)

また、上記のコードは現状コレ↓と同等だけど後々（14章）の為にこの↑形に
```rb
def feed
  microposts
end
```  


# フィード機能の実装
```rb
# 以前のコードが
@micropost = current_user.microposts.build if logged_in?
 
↓
 
# こうなっている
if logged_in?
  @micropost  = current_user.microposts.build
  @feed_items = current_user.feed.paginate(page: params[:page])
 end
```

1行のときは後置if文、
2行以上のときは前置if文を使うのがRubyの慣習

## フィードのパーシャル
```rb
<%= render @feed_items %>
```

@feed_itemsにはcurrent_userに紐付いたfeedのpaginate(page: params[:page])が代入されており

```rb
# @feed_itemsにcurrent_userに紐付いたfeedのpaginate(page: params[:page])を代入
@feed_items = current_user.feed.paginate(page: params[:page])
```

feedってなんだっけって言うと

```rb
def feed
  # Micropostテーブルからuser_idがidのユーザーをすべて取得
  Micropost.where("user_id = ?", id)
end
```

つまり@feed_itemsの各要素はMicropostクラスを持っている
よってRailsはMicropostのパーシャル（_micropost.html.erb）を呼び出すことができる。


## request.referrerメソッド

```request.referrer || root_url```
このメソッドは一つ前のURLを返します
フレンドリーフォワーディングのrequest.url変数 (10.2.3) と似てる！


## redirect_backメソッド

```redirect_back(fallback_location: 例外のリダイレクト先)```

```rb
flash[:success] = "Micropost deleted"
    # リダイレクト (request.referrer で返される)1つ前のurl またはroot_url
    request.referrer || root_url
    # redirec_backで直前に実行したアクションへリダイレクト
    # 引数のfallback_locationオプションで例外が発生したときroot_urlにリダイレクト
    # redirect_back(fallback_location: root_url)
```

# 13.4.1 演習 解説
```rb:/sample_app/test/integration/microposts_interface_test.rb
# 特定のhtmlタグが存在する type=“file”という条件付きのinputタグ
assert_select ‘input[type=“file”]’
```

## input type="file" とは
type="file" 型の <input> 要素は、ユーザーが一つまたは複数のファイルを端末のストレージから選択することができるようにします。
選択されると、ファイルはフォーム投稿を使用してサーバーにアップロードしたり、 JavaScript コードと File API を使用して操作したりすることができます。

cf. [input type="file"](https://developer.mozilla.org/ja/docs/Web/HTML/Element/Input/file)

cf. [【Railsメモ】assert_selectについて](https://qiita.com/TakahitoNakashima/items/b439126cd1ee302586d7#assert_select%E3%81%A8%E3%81%AF)
