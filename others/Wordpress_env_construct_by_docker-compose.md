# はじめに

【この記事は**5分**で読了できます】

docker-composeを使ってWordPressの開発環境をたった3分で爆速構築します。
(まだpullされていないimageをpullする時間は除きます。)

今回はDockerの[公式ドキュメント](https://docs.docker.com/compose/wordpress/)(英語)を参考に進めますので、英語のドキュメントが読める方はそちらをご参考ください。

## 動作確認環境

- macOS Catalina (ver. 10.15.5)
- Docker Desktop for Mac (ver. 2.3.0.3)

## 手順

実際にやっていきましょう！

### 1. ディレクトリの作成

新しくディレクトリを作成します。
今回は[my_wordpress]という名前にしました。

```console
mkdir my_wordpress
cd my_wordpress/
```

### 2. docker-compose.ymlの作成

先ほど作ったディレクトリ直下にdocker-compose.ymlファイルを作ります。

```console
touch docker-compose.yml
```

docker-compose.ymlに以下の内容を記述します。

```yml:docker-compose.yml
version: '3.3'

services:
   db:
     image: mysql:5.7
     volumes:
       - db_data:/var/lib/mysql
     restart: always
     environment:
       MYSQL_ROOT_PASSWORD: somewordpress
       MYSQL_DATABASE: wordpress
       MYSQL_USER: wordpress
       MYSQL_PASSWORD: wordpress

   wordpress:
     depends_on:
       - db
     image: wordpress:latest
     ports:
       - "8000:80"
     restart: always
     environment:
       WORDPRESS_DB_HOST: db:3306
       WORDPRESS_DB_USER: wordpress
       WORDPRESS_DB_PASSWORD: wordpress
       WORDPRESS_DB_NAME: wordpress
volumes:
    db_data: {}
```

### ビルド

docker-composeでビルドします。
detachedモードで起動する```-d```オプションをつけています。

```console
docker-compose up -d
```

### WordPressの起動

ブラウザでWordPressを起動します。
Docker Desktop for Mac (or Windows)の場合、<http://localhost:8000>にアクセスします。

### 言語選択

アクセスすると以下の画面になります。
お好きな言語を選択してください。（日本語もあります。）
![wordpress-lang.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/235035/53ee92c9-fa92-b4f2-f1fb-315ff1876758.png)

### フォーム入力

次に以下の画面になります。
必要な情報をフォームに入力し、Install WordPressをクリックします。
![wordpress-welcome.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/235035/5670a804-79a7-bb0c-6733-a5d88e49c2fb.png)

### WordPress開発環境構築

無事WordPress開発環境が構築できました！やったー！

### 終了コマンド

以下のコマンドで終了できます。

```docker-compose down```
コンテナとデフォルトのネットワークを削除しますが、WordPressのデータベースはそのまま保持されます。

```docker-compose down --volumes```
コンテナとデフォルトのネットワークを削除し、WordPressのデータベースも削除します。

## 参考サイト

cf. https://docs.docker.com/compose/wordpress/
