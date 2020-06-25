# Docker コマンド
- run = create + start

- $ Docker rm
コンテナを削除

- $ Docker run --name <container_name> <image>
コンテナ名を指定

- $ Docker system prune
停止している全てのコンテナを一括削除

- $ Docker run -d <image>
コンテナ起動後にdetachする（バックグラウンドで動こかす）

- $ Docker run --rm <image>
コンテナexit後に削除する

# Dockerfile
Docker image の設計図のようなもの
DockerfileからDocker image を作る

## Docker buildコマンド
```docker build ```
DockerfileからDocker image を作成

```docker build -t <name> <directory>```
ID名を指定してbuild

## Dockerfile記法
**Docker imageのLayer数は最小限にする！**
- コマンドを&&でつなげる
- バックスラッシュで改行

### FROM
ベースとなるイメージを決定

E.g. Ubuntu, Alpine(5MBしかなくてコンパクト）

### RUN
Linuxコマンドを実行

```
apt-get update
apt-get install
```

### CMD
コンテナのデフォルトのコマンドを指定
```CMD [ “executable”, “param1”, “param2”]```
原則Dockerfileの最後に1回だけ記述

RUNはLayerを作る
CMDは作らない

### COPY
```COPY <sorce> <destination>```
ホストからコンテナにファイルをコピーする

### ADD
```ADD <sorce> <destination>```
ほとんどがtarファイル（サイズが大きいものを圧縮したりできる）を追加する時に使う

### COPY vs ADD
- 単純にファイルやフォルダをコピーする場合はCOPY
- tarの圧縮ファイルをコピーして解答したいときはADD

### Dockerfileという名のファイルがビルドコンテキストに入っていない場合
```docker build -f <Dockerfile-name> <build-context>```

本番環境とテスト環境でDockerfileを分ける場合になどに使う

e.g. Dockerfile.dev と Dockerfile.test 

### ENTRYPOINT
- ENTRYPOINTはrun時に上書きできない
- ENTRYPOINTがある場合はCMDは[“params1”, “params2”]の形をとる（つまり、ENTRYPOINTで指定したコマンドの引数）
- Run時に上書きできるのはCMD部分のみ
- コンテナをコマンドのように使いたい時に使う

# Docker run オプション
## -v
-v <host>:<container> 
ホストのファイルシステムをコンテナにマウントする（実行環境で使う）
コンテナにディレクトリがない時は自動で新規作成される

## -u
-u <user-id>:<group-id>
原則以下の形で権限を指定
```-u $(id -u):$(id -g)```
ユーザーIDとグループIDを指定してコンテナをrunする

## -p
-p <host_port>:<container_port>
ホストのポートをコンテナのポートにつなげる
E.g. ```-p 8888:8888```
ポート
プロセスがデータ通信するために使うもの

## --cpus --memory
```--cpus <#ofCPUs> --memory <byte>```
コンピューターリソースの上限の設定

## inspect コマンド
```docker inspect <container> | grep -I <something>```
コンテナの詳細を表示

