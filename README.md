# docker-hands-on

## キッカケ

プロダクト内の勉強会で Docker の話をしたら「試してみたい！」というメンバーが多かった．

教えられるほど詳しくないけど，Docker の基礎の部分をハンズオン形式でやったら面白そうってことで教材を雑に書いている．

* [Docker Tシャツを着て Docker の話をした - kakakakakku blog](http://kakakakakku.hatenablog.com/entry/2015/08/08/000149)

## 目的

Docker の基礎を実際に手を動かしながら理解する．

## ゴール

2時間で試せる範囲という意味で今回のゴールを以下のように定める．

* Docker のコマンドの意味を理解して叩けること
* 手動で構築を構築して自分用のイメージにできること
* Dockerfile を使ってイメージを作れること
* Docker Hub に自分用のイメージを公開できること

## boot2docker on Mac

まず Mac に boot2docker をインストールする．バージョンは `v1.7.1` にする．

インストール後に `boot2docker version` が叩けるようになっていれば OK と言える．

* [Boot2docker by boot2docker](http://boot2docker.io/)
* [boot2docker/osx-installer](https://github.com/boot2docker/osx-installer)

```sh
➜  ~  boot2docker version
Boot2Docker-cli version: v1.7.1
Git commit: 8fdc6f5
```

## boot2docker を起動する

Docker で使う環境変数を設定した状態で起動する．

```sh
➜  ~  eval "$(boot2docker shellinit)"
➜  ~  boot2docker up
```

以下のコマンドが返ってきたら正常に起動している．

```sh
➜  ~  docker info
```

## CentOS の Docker イメージを取得する

`docker images` で結果が表示されたら正常に取得できている．

```sh
➜  ~  docker images
➜  ~  docker pull centos:centos6
➜  ~  docker images
REPOSITORY           TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
centos               centos6             a005304e4e74        7 weeks ago         203.1 MB
```

## コンテナを起動して httpd をインストールする

コンテナ内のオペレーションは PS1 が `[]` になっている．

httpd を起動してデフォルトページが返ってくるのを確認できたら正常にインストールできている．

```sh
➜  ~  docker run -it centos:centos6 /bin/bash

[root@c0a5325eebaf /]# yum install -y httpd
（中略）
Complete!
[root@c0a5325eebaf /]# service httpd start
[root@c0a5325eebaf /]# curl http://localhost
```

ここで別のターミナルを開いて `docker ps` と `docker ps -a` を叩いてみる．

現在はコンテナが稼働しているため，どちらのコマンドを使っても，コンテナプロセスを確認することができる．

```
➜  ~  docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
c0a5325eebaf        centos:centos6      "/bin/bash"         7 minutes ago       Up 7 minutes                            goofy_bohr

➜  ~  docker ps -a
CONTAINER ID        IMAGE                COMMAND                CREATED             STATUS                     PORTS               NAMES
c0a5325eebaf        centos:centos6       "/bin/bash"            7 minutes ago       Up 7 minutes                                   goofy_bohr
```

## コンテナを停止する

コンテナを停止する．

```sh
[root@c0a5325eebaf /]# exit
exit
```

停止したらもう1度 `docker ps` と `docker ps -a` を叩いてみる．

オプションを付けないと稼働中のコンテナだけが表示されることがわかる．

* [ps](https://docs.docker.com/reference/commandline/ps/)

```sh
➜  ~  docker ps
dCONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES

➜  ~  docker ps -a
CONTAINER ID        IMAGE                COMMAND                CREATED             STATUS                          PORTS               NAMES
c0a5325eebaf        centos:centos6       "/bin/bash"            10 minutes ago      Exited (0) About a minute ago                       goofy_bohr
```

## httpd コンテナをイメージにする

イメージ名は任意で OK だけど `${USERNAME}/${IMAGENAME}` にする慣習に沿う．

```sh
➜  ~  docker commit c0a5325eebaf kakakakakku/manual-httpd
4021b02c4720fc422d14ca7477121586d006dbc5f48faa0e74c607f5f01d09b5

➜  ~  docker images
REPOSITORY                 TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
kakakakakku/manual-httpd   latest              4021b02c4720        13 seconds ago      257.5 MB
centos                     centos6             a005304e4e74        7 weeks ago         203.1 MB
```

## httpd イメージを Mac にポートマッピングをした状態で起動する

Mac の 8080 ポートを Docker コンテナの 80 ポートにマッピングした状態で，既に作った httpd イメージを起動する．

この時点では httpd は落ちているため起動する．

```
➜  ~  docker run -p 8080:80 -it kakakakakku/manual-httpd /bin/bash
[root@9b4ad6f91ea7 /]# service httpd start
```

別のターミナルを開いて Docker が起動してる IP を確認しておく．

```sh
➜  ~  boot2docker ip
192.168.59.103
```

Chrome などのブラウザで `http://192.168.59.103:8080` にアクセスして接続できることを確認する．

## コンテナの生存期間を体験する

コンテナ上で `Control + p, Control + q` と入力するとターミナルからデタッチすることができる．

普通に `exit` をしてしまうとコンテナが停止してしまうが，デタッチするとコンテナは稼働したままになる．

```sh
➜  ~  docker ps
CONTAINER ID        IMAGE                      COMMAND             CREATED             STATUS              PORTS                  NAMES
9b4ad6f91ea7        kakakakakku/manual-httpd   "/bin/bash"         6 minutes ago       Up 6 minutes        0.0.0.0:8080->80/tcp   grave_bell
```

この状態でブラウザから接続しても，引き続き `http://192.168.59.103:8080` にアクセスできることがわかる．

次にコンテナを外から停止してみる．

```sh
➜  ~  docker stop 9b4ad6f91ea7
9b4ad6f91ea7
```

するとコンテナが停止するため，アクセスできなくなる．

この一連の流れで，コンテナの生存期間を体験することができたと思う．
