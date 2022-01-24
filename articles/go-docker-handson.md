---
title: "Go と Docker ハンズオン" # 記事のタイトル
emoji: "😸" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["Go", "Docker"] # タグ。["markdown", "rust", "aws"]のように指定する
published: true # 公開設定（falseにすると下書き）
---
# はじめに

- 最近、コンテナと呼ばれる技術により Docker や Kubernetes が注目されている
- 複数台のサーバーに、簡単にアプリケーションを展開できるから
- コンテナでは Go や Rust のような比較的軽量のシングルバイナリが起動時間の観点から好まれている

すでに Docker は普及しているが、今後も発展していく。相性の良い Go なども発展していくことが予想できるため、早いうちにキャッチしておこう！

# Web アプリケーションの潮流

Kubernetes などが急速に広まる状況で想像することは難しいと思うが、歴史について触れておきます。


## 物理サーバー

インフラといえば、アプリケーションがどれくらい利用されるか予測し、物理サーバを業者に発注していました（調達と言います）。メーカの都合でハードウェアや仕様が変わったり、求めるサーバーが手に入らないことがあるわけです。

負荷対策では強いマシンを用意したり、ロードバランサーと複数台のマシンで分散処理します。


## 複数サーバーへのデプロイ

サーバーが少数台であれば、1台1台にアプリケーションコードを配置（デプロイメント）するのは難しくありません。

rsync などのツールを使って効率的にコードを配置していました。

しかし、サーバー台数が多くなってきたり、さまざまなサーバー（仕様やOS）が混在するようになると難度は上がっていきます。セットアップにミスがあると、サーバーによっては Python が入っていないだとか、バージョンが古いだとか、トラブルを引き起こします。

こうしたサーバ環境を管理することをプロビジョニングと言います。Puppet、Ansible、Chef がそれらツールの代表です。サーバーに一括で同じスクリプトを実行することが簡単になりました。


## 仮想化技術

データセンターでは、サーバー仮想化が普及することになります。1台のつよつよマシンを、複数台の仮想サーバとして分割するための技術です。

ホストマシンの中で OS とカーネルを仮想化し、ユーザにアプリケーション実行環境を公開します。レンタルサーバの裏側で使われている技術です。

この仮想化技術を使って大量のサーバを保持する会社が、指定したサーバスペックを短時間で調達する IaaS が生まれました。仮想化によってサーバ環境の統一やプロビジョニングが容易になりました。


## Docker の誕生

先程述べた仮想化技術はスーパーバイザ型、ホスト型などと呼ばれ、ホストマシン上でゲストマシンを立ち上げ、 OS やカーネル、アプリケーションを動かします。

これらの技術を使うと、サーバ調達は短時間で簡単に済みます。とはいえ、もっと素早いリソースの調整が求められていました。アプリケーションが主体になっていく流れの中で OS やカーネルまで仮想化が必要なのかという疑問が生まれました。


アプリケーション動作環境をうまく仮想化するだけの技術として Docker が生まれました。 Docker はホストマシンのOSとカーネルを共有し、アプリケーションだけを動作させます。

OSやカーネルを仮想化しない分、短時間で起動が可能です。もともと Linux の namespace と LXC を使っていましたが、いまでは Linux ディストリビューションに限らず Window や macOS などで使えるような仕組みに変わっています。


## コンテナオーケストレーション

Docker はコンテナと呼ばれる隔離環境でアプリケーションを実行します。1台のサーバ上でお互いのプロセスに影響を与えることなく、複数のコンテナを動かすことが可能になります。

となると、どのサーバーにどのコンテナを動かすのかという問題が生じます。これを解決するのが、コンテナオーケストレーションツールです。 Kubernetes や Apache Mesos がそれらにあたります。Apache Mesos は Docker 以前から存在していました。

このようにして簡単に生成・破棄できるコンテナ型仮想化技術が重宝され、新しくオーケストレーションの問題が生まれ、また新しく Kubernetes のようなツールで解決が図られているという流れです。


# Docker

さて、前置きは以上として Docker を試してみましょう。 まずは whalesay というコマンドを Docker 経由で利用します。

docker run によって Docker Container を起動できます。

```shell
docker run docker/whalesay cowsay "Hello World!"
```

docker/whalesay は Docker Image の名前です。その後続がコマンドです。`Hello World!` は好きな文字を入れましょう。


## Docker Container の中でシェルを使う

docker/whalesay は Bash を持っているのでシェルを利用することができます(`-it` で tty を有効化するのを忘れずに)。

```shell
docker run -it docker/whalesay bash
```

試しに OS の情報を参照しましょう。 Ubuntu 14.04.2 LTS であることがわかります。古いですね。いまは 21.04 がリリース済みですし、 20.04 が LTS です。古い環境もすぐに試せるのが Docker の良いところです。

```
root@3b952dcabf0b:/cowsay# cat /etc/os-release 
NAME="Ubuntu"
VERSION="14.04.2 LTS, Trusty Tahr"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 14.04.2 LTS"
VERSION_ID="14.04"
HOME_URL="http://www.ubuntu.com/"
SUPPORT_URL="http://help.ubuntu.com/"
BUG_REPORT_URL="http://bugs.launchpad.net/ubuntu/"
```

Ctrl+D で抜けておきましょう。


## Docker 概要

Docker Image から Docker Container が生成されます。実際に動作するのは Docker Container です。

`docker run docker/whalesay` によって Docker は docker/whalesay というイメージを探します。今回はローカルに存在しなかったので DockerHub で探してダウンロードされました。 イメージをダウンロードするだけなら `docker pull` を使います。

ダウンロードした Docker Image は 次のコマンドで閲覧できます。

```shell
docker image ls
```

Docker Container の状況は次のコマンドで閲覧できます( `-a` はストップしているものも表示するオプションです)。
`docker/whalesay` から作られたコンテナは、コマンド実行後ストップします。

```shell
docker container ls -a
```

Web アプリケーションの場合は、ストップしないことが多いので `-a` は不要です。


## 注意

Docker Image や Docker Container はローカルコンピューターの中に蓄積していくのでストレージ容量を圧迫します。たとえば `docker/whalesay` イメージは 247MB の容量があります。 Docker Image には OS や カーネルを含んでいないのでほかのスーパーバイザー型仮想化技術よりも軽量ですが、実行に必要なライブラリやアプリケーションが含まれているためです。

```
$ docker image ls | grep whale
docker/whalesay                  latest               6b362a9f73eb   6 years ago     247MB
```

不要な Docker Image や Container は定期的に削除しましょう。

実際に削除します(失敗します)。

```
$ docker image rm docker/whalesay
Error response from daemon: conflict: unable to remove repository reference "docker/whalesay" (must force) - container 6bed5286f103 is using its referenced image 6b362a9f73eb
```

エラーを読むと Container から削除が必要です。 Container のほうから削除しましょう

Container を削除するには Container ID か name が必要です。 `docker container ls` で参照しましょう

```
$ docker container ls -a --filter status=exited
CONTAINER ID   IMAGE             COMMAND                  CREATED          STATUS                       PORTS     NAMES
3b952dcabf0b   docker/whalesay   "bash"                   9 minutes ago    Exited (127) 4 minutes ago             objective_bhaskara
12e392122dcf   docker/whalesay   "cowsay 'Hello World…"   33 minutes ago   Exited (0) 33 minutes ago              optimistic_brahmagupta
6bed5286f103   docker/whalesay   "cowsay boo"             34 minutes ago   Exited (0) 34 minutes ago              fervent_curran
```

`rm` で削除できます。複数指定できるので、複数出てきた人は複数指定しましょう。

```shell
docker container rm 3b952dcabf0b 12e392122dcf 6bed5286f103
```

そして `docker image rm` で `docker/whalesay` 削除します。

```
$ docker image rm docker/whalesay
Untagged: docker/whalesay:latest
Untagged: docker/whalesay@sha256:178598e51a26abbc958b8a2e48825c90bc22e641de3d31e18aaf55f3258ba93b
Deleted: sha256:6b362a9f73eb8c33b48c95f4fcce1b6637fc25646728cf7fb0679b2da273c3f4
Deleted: sha256:34dd66b3cb4467517d0c5c7dbe320b84539fbb58bc21702d2f749a5c932b3a38
Deleted: sha256:52f57e48814ed1bb08a651ef7f91f191db3680212a96b7f318bff0904fed2e65
Deleted: sha256:72915b616c0db6345e52a2c536de38e29208d945889eecef01d0fef0ed207ce8
Deleted: sha256:4ee0c1e90444c9b56880381aff6455f149c92c9a29c3774919632ded4f728d6b
Deleted: sha256:86ac1c0970bf5ea1bf482edb0ba83dbc88fefb1ac431d3020f134691d749d9a6
Deleted: sha256:5c4ac45a28f91f851b66af332a452cba25bd74a811f7e3884ed8723570ad6bc8
Deleted: sha256:088f9eb16f16713e449903f7edb4016084de8234d73a45b1882cf29b1f753a5a
Deleted: sha256:799115b9fdd1511e8af8a8a3c8b450d81aa842bbf3c9f88e9126d264b232c598
Deleted: sha256:3549adbf614379d5c33ef0c5c6486a0d3f577ba3341f573be91b4ba1d8c60ce4
Deleted: sha256:1154ba695078d29ea6c4e1adb55c463959cd77509adf09710e2315827d66271a```
```


一括削除には `docker image prune` や `docker container prune` が便利です（安易に実行しないようにしてください）。


# Dockerfile

`docker/whalesay` という Docker Image を使いました。この Docker Image は自作できます。そのためのファイルが Dockerfile です。
自作した Docker Image は DockerHub やオープンソースの [Harbor](https://goharbor.io/), Google Cloud Registry, Amazon ECR などに公開可能です(コンテナレジストリと言います)。

Dockerfile を用意する準備をしましょう。 `cat` するだけの Docker Image を作ります。

```
$ mkdir dockerfile-trial
$ cd !$
$ touch Dockerfile
$ echo "word" > word
```

参考: https://qiita.com/zembutsu/items/24558f9d0d254e33088f

```Dockerfile
FROM debian:bullseye-slim

WORKDIR /app/
COPY ./word /app/

CMD ["cat", "./word"]
```

`FROM` インストラクションは Docker Image のベースを指定します。まったくなしの状態であれば `scratch` を使います。有名な Linux ディストリビューションである CentOS や Ubuntu などのベースイメージも用意されていますが、アプリケーション実行環境にしては不要なファイルが多くストレージ容量を圧迫しがちなので避けたほうが良いです。


`COPY` というのはローカルのファイルをコピーするインストラクションです。 似たようなインストラクションとして `ADD` というものがありますが、こちらは URL から tar を取得し展開までやってくれます。一見便利ですが、セキュリティ的な懸念があるので単なるコピーでは使わないようにしてください。代わりに `COPY` を使いましょう。

では Docker Image を作るためにビルドします。
`docker build` でビルドできます。 `-t` によって `cat-word` と名付けています。最後の `.` はコンテキストといってどのディレクトリを基準にするか指定しています。

```
$ docker build -t cat-word .
Sending build context to Docker daemon  134.9MB
Step 1/4 : FROM debian:bullseye-slim
bullseye-slim: Pulling from library/debian
a2abf6c4d29d: Already exists 
Digest: sha256:b0d53c872fd640c2af2608ba1e693cfc7dedea30abcd8f584b23d583ec6dadc7
Status: Downloaded newer image for debian:bullseye-slim
 ---> 8bca477113bd
Step 2/4 : WORKDIR /app/
 ---> Running in 67651f053456
Removing intermediate container 67651f053456
 ---> aa1cce7602e7
Step 3/4 : COPY ./word /app/
 ---> 58c9706d354b
Step 4/4 : CMD ["cat", "./word"]
 ---> Running in dd246fadf91b
Removing intermediate container dd246fadf91b
 ---> d383a2df6024
Successfully built d383a2df6024
Successfully tagged cat-word:latest
```

成功したようですね。実行します。

さきほど `docker container rm` で Docker Container を削除しました。実行するたびにコンテナが残るのは面倒なので `--rm` オプションによって Docker Container が停止したら削除するように指示しておきます(Docker Image は削除されません)

```
$ docker run --rm cat-word
hello
```

`word` というファイルの中身と同じものが出力されたはずです。ファイルを書き換えてビルドし直せば結果が変わります。

```shell
echo "nyan" > word
docker build -t cat-word .
```

Docker build は結果をキャッシュしているので、ビルドがさきほどよりも短時間で終わります。

```
$ docker run --rm cat-word
nyan
```

ここまでで Docker Image を自作し Docker Container を実行する流れを学びました。


# Go

Go は学習コストが低く、リーダブルなコードが書けるプログラミング言語です。 Go はクロスプラットフォームのコンパイルが可能ですし、比較的軽量なシングルバイナリを生成します。

メモリ管理の手間が少ないというメリットがありますが GC のアルゴリズムが STOP THE WORLD を採用しているため、ミッションクリティカルな場面やメモリが貧弱な環境には適しません。

Go のコードはセルフホスティングされているので、 Go のコードをよりうまく書きたいなら Go 内部のコードを読むこともおすすめです。 https://pkg.go.dev/ にて Go のドキュメントを閲覧できます。

実際に Go を書いていきましょう。 Go のプロジェクトを始めるときは `go mod init` から始めます。

```shell
mkdir gohandson
go mod init go-docker-handson
```

昔は GOPATH や GOROOT という環境変数に依存していましたが、いまでは気にする必要はありません。

まずは Hello World からです。 `main.go` というファイルを作ります。

```go
package main

import "fmt"

func main() {
	fmt.Println("hello")
}
```

`go run main.go` で実行できます。

```
$ go run main.go 
hello
```

Go の関数や変数は、大文字からスタートするとほかのパッケージから参照できるようになります。小文字だと参照できません。

`go build` によってシングルバイナリを作成できます。

```shell
go build main.go -o main
```

Go はエントリポイントを cmd ディレクトリに下に作ることが多いです。それに則りましょう。

```shell
mkdir -p cmd/cli
mv main.go cmd/cli
```

少々拙速ですが、次に HTTP API を作っていきます。 Go は net/http という標準パッケージが強力なのでフレームワークなしでもそこまで困りません。

今回は簡単のため、 [echo](https://echo.labstack.com/) を使います。 インストールは `go get` で行えます。

```shell
go get github.com/labstack/echo/v4
```

```shell
mkdir cmd/api
touch cmd/api/main.go
```

`cmd/api/main.go` には次のように記述しましょう。

```go
package main

import (
	"net/http"

	"github.com/labstack/echo/v4"
)

func main() {
	e := echo.New()  // echo を利用する
    // GET リクエストでパスが `/` のとき第２引数の関数を実行する
	e.GET("/", func(c echo.Context) error {
		return c.String(http.StatusOK, "Hello, World!")
	})

    // 1323 ポートでリッスンを開始。 start がエラーを起こしたら Fatal を起こしてログに記録する
	e.Logger.Fatal(e.Start(":1323"))
}
```

書き終わったら実行します。Web サーバーがリッスンを始めるので localhost:1323 にアクセスして Hello World を確認しましょう。

```shell
go run cmd/api/main.go
```

# Dockerfile で Go のアプリケーションをコンテナ化

これを Dockerfile に起こしましょう。 `docker/app` というディレクトリを作って Dockerfile を置きます。

```shell
mkdir -p docker/app
touch docker/app/Dockerfile
```

```Dockerfile
FROM golang:1.17.6 as builder
WORKDIR /workspace
COPY . /workspace
# alpine でも実行できるように GOOS と CGO_ENABLED を指定
RUN CGO_ENABLED=0 GOOS=linux go build -o main cmd/api/main.go && chmod +x ./main

FROM alpine:3.15
WORKDIR /app
RUN apk --no-cache add ca-certificates
# root ユーザだとなんでもできてしまうため appuser を作成する
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
COPY --from=builder /workspace/main ./
# コンテナを立ち上げたとき、勝手にWeb サーバーを立ち上げる
CMD ["./main"]
```

上記の Dockerfile には FROM インストラクションが二度現れています。これはビルダーパターンと呼ばれていて、コンパイルする環境と実行する環境を分ける Dockerfile のパターンです。
コンパイルするときは、さまざまなライブラリが必要ですが実行時には不要な場合が多いです。セキュリティを堅牢にしたり、イメージの容量を圧縮するために用いられます。

Dockerfile が書けたらビルドを実行します。今回は myapp という名前でコンテナを作ります。

```shell
docker build -t myapp -f docker/app/Dockerfile .
```

※作成された Docker Image がどれくらいの容量になっているか確認してみてください(コマンドは示しません)。

起動を確認しましょう。 `-p` はコロンを堺に左がホスト側、右がコンテナ側のポートをマップするオプションです。

```shell
docker run -it -p1323:1323 --rm myapp
```

`localhost:1323` にアクセスすると echo が動いていることが確認できます。


# Docker Compose

Docker はコンテナを操作するためのコマンドでした。一般的な Web サービスだと、コンテナは1つでは足りません。アプリケーション、 Web サーバー、キュー、データベースなどさまざまなコンテナが必要です。それを一手に管理してくれるのが docker-compose です。

docker-compose.yaml というファイルを作ります。

```yaml
version: "3"
services: 
  app: # サービスの名前
    build:
      context: .
      dockerfile: docker/app/Dockerfile
    tty: true
    ports:
      - 1323:1323
    volumes:
      - .:/go/src/app
```

`docker-compose build` によって簡単にビルドができます。 `docker-compose up` するとサーバーが立ち上がります

実際の開発では docker-compose が非常に便利なので基本的には docker コマンドは使いません。


# カウンターを作る

Redis を使ってアクセスカウンターを作ってみましょう。 Go から Redis を利用するために redigo を利用します。

Redis はインメモリ型のデータストアです。データベースやキャッシュ、メッセージブローカーとして利用されます。アトミック処理やトランザクション処理もサポートしているため、カウンターのような複数アプリケーションから同時アクセスのあるアプリケーションに適しています。データ構造の関係上、単純なクエリしかサポートしていないため、検索用途には向きません。


## redigo のインストール

`go get` で redigo をインストールします。

```shell
go get github.com/gomodule/redigo/redis
```

アプリケーションの変更は次のコミットを参考にしてください。

https://github.com/tamanobi/go-docker-handson/commit/a22930a8feacd29a7d73f2a60c4d1f6163e37f80

