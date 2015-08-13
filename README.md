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
* 手動でコンテナを構築しイメージを作れること
* Dockerfile を使ってイメージを作れること
* Docker Hub にイメージを公開できること

## 1. 環境準備

### 1-1. boot2docker on Mac

まず Mac に boot2docker をインストールする．バージョンは `v1.7.1` にする．

インストール後に `boot2docker version` が叩けるようになっていれば OK と言える．

* [Boot2docker by boot2docker](http://boot2docker.io/)
* [boot2docker/osx-installer](https://github.com/boot2docker/osx-installer)

```sh
➜  ~  boot2docker version
Boot2Docker-cli version: v1.7.1
Git commit: 8fdc6f5
```

### 1-2. boot2docker を起動する

Docker で使う環境変数を設定した状態で起動する．

```sh
➜  ~  boot2docker init
➜  ~  boot2docker up
➜  ~  eval "$(boot2docker shellinit)"
```

以下のコマンドが返ってきたら正常に起動している．

```sh
➜  ~  docker info
```

## 2. 手動でコンテナを構築する

### 2-1. CentOS の Docker イメージを取得する

`docker images` で結果が表示されたら正常に取得できている．

```sh
➜  ~  docker images
➜  ~  docker pull centos:centos6
➜  ~  docker images
REPOSITORY           TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
centos               centos6             a005304e4e74        7 weeks ago         203.1 MB
```

### 2-2. コンテナを起動して httpd をインストールする

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

### 2-3. コンテナを停止する

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

### 2-4. httpd コンテナをイメージにする

イメージ名は任意で OK だけど `${USERNAME}/${IMAGENAME}` にする慣習に沿う．

```sh
➜  ~  docker commit c0a5325eebaf kakakakakku/manual-httpd
4021b02c4720fc422d14ca7477121586d006dbc5f48faa0e74c607f5f01d09b5

➜  ~  docker images
REPOSITORY                 TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
kakakakakku/manual-httpd   latest              4021b02c4720        13 seconds ago      257.5 MB
centos                     centos6             a005304e4e74        7 weeks ago         203.1 MB
```

### 2-5. httpd イメージを Mac にポートマッピングをした状態で起動する

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

### 2-6. コンテナの生存期間を体験する

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

## 3. Dockerfile を使ってイメージを作る

僕が作ってる Dockerfile は以下のリポジトリにまとめている．

* [Kakakakakku/dockerfiles](https://github.com/Kakakakakku/dockerfiles)

### 3-1. httpd コンテナを構築する Dockerfile を作成する

`my_docker` のように任意のディレクトリを作成して `Dockerfile` を作成する．

```sh
➜  my_docker  vim Dockerfile
```

各項目の詳細はリファレンスに書いてある．

* [Dockerfile reference](https://docs.docker.com/reference/builder/)

```
FROM centos:centos6
MAINTAINER Yoshiaki Yoshida <y.yoshida22@gmail.com>

RUN yum install -y httpd
EXPOSE 80
CMD ["/usr/sbin/httpd", "-D", "FOREGROUND"]
```

### 3-2. Dockerfile からイメージを構築する

```sh
➜  my_docker  docker build -t kakakakakku/httpd .
（中略）
Successfully built 0507c25b6917
```

正常にビルドが完了するとイメージが構築されていることがわかる．

```sh
➜  my_docker  docker images
REPOSITORY                 TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
kakakakakku/httpd          latest              0507c25b6917        20 seconds ago      257.5 MB
```

### 3-3. 構築したイメージを起動する

今回はバックグラウンド起動を意味した `-d` をオプションで起動する．

```sh
➜  my_docker  docker run -p 8080:80 -d kakakakakku/httpd
8096ff4df716407d554896b4590644530ec29e0e7225f5126001578ffc9f3547
```

この状態で既に `http://192.168.59.103:8080` にアクセスできるようになっている．

先ほどと同様にコンテナを外から停止する．

```sh
➜  my_docker  docker ps
CONTAINER ID        IMAGE               COMMAND                CREATED             STATUS              PORTS                  NAMES
8096ff4df716        kakakakakku/httpd   "/usr/sbin/httpd -D    2 minutes ago       Up 2 minutes        0.0.0.0:8080->80/tcp   furious_torvalds

➜  my_docker  docker stop 8096ff4df716
8096ff4df716
```

## 4. Docker Hub にイメージを公開する

### 4-1. Docker Hub のアカウントを取得する

以下で取得できる．個人のページを見ることもできる．

* [Docker Hub](https://hub.docker.com/)
* [Docker Hub](https://hub.docker.com/u/kakakakakku/)

### 4-2. Docker Hub に push する

対象のイメージを確認しておく．

```sh
➜  my_docker  docker images
REPOSITORY                 TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
kakakakakku/httpd          latest              0507c25b6917        7 minutes ago       257.5 MB
```

`docker push` で Docker Hub に登録することができる．

初回はアカウント情報を対話的に求められるが，2回目以降は `~/.docker/config.json` にファイルがあるため求められなくなる．

```sh
➜  my_docker  docker push kakakakakku/httpd

Please login prior to push:
Username:
Password:
Email:
Login Succeeded
（中略）
Digest: sha256:6f4444069187b4e10d5b11694b86fd6badc035f8cb3dc7b16e85566d710c65db
```

自分の Docker Hub ページを見て正常に push できていることを確認する．

# まとめ

Docker 超入門お疲れさまでした！
