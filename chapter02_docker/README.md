# Docker


このセクションでは、Dockerを使ってコンテナアプリケーションの基本的な作成方法やdockerコマンドの使用方法を学びます。
尚、このセクションではご自身のDocker Hub及びプライベートリポジトリを利用します。
未作成の方はサインアップ及びプライベートリポジトリの作成をお願いします。
https://hub.docker.com/signup

## 1. コンテナ作成



まず、以下のコマンドでDockerの正常性確認を行います。



```Bash
docker version
# or
docker -v
```


```
# 実行結果
Docker version 26.0.0, build 2ae903e
```

その後、自身のDocker Hubにログインを行います。


```
docker login
```


続いて、サンプルイメージをpullします。

このイメージはDocker社が作成したチュートリアルのWebアプリケーションです。


```
docker pull docker/getting-started
```


```
# 実行結果
Using default tag: latest
latest: Pulling from docker/getting-started
df9b9388f04a: Pull complete 
5867cba5fcbd: Pull complete 
4b639e65cb3b: Pull complete 
061ed9e2b976: Pull complete 
bc19f3e8eeb1: Pull complete 
4071be97c256: Pull complete 
79b586f1a54b: Pull complete 
0c9732f525d6: Pull complete 
Digest: sha256:b558be874169471bd4e65bd6eac8c303b271a7ee8553ba47481b73b2bf597aae
Status: Downloaded newer image for docker/getting-started:latest
docker.io/docker/getting-started:latest
```


以下のコマンドで正常にPullができたか確認が行えます。


```Bash
docker image ls
# or
docker images
```


```
# 実行結果
docker/getting-started                                  latest    cb90f98fd791   4 months ago    28.8MB
```


続いて、以下のコマンドでサンプルコンテナを起動します。

-dはバックグラウンド動作、-pはPortforwardの設定オプションです。


```Bash
docker run -d -p 8888:80 docker/getting-started
```


```
# 実行結果
fe5facdead0cc4645abf79f477c44d8a5d99690e4478942e9c56cb7959fc5201
```


アクセス確認を行います。

正常にコンテナが起動できていれば、ステータスコード200が返却されるはずです。


```
curl -I localhost:8888
```


次に、コンテナを停止します。


コンテナを停止するにはコンテナIDが必要なため
以下のいづれかのコマンドで自身のコンテナIDを出力します。


```Bash
docker container ls | grep getting-atarted
# or
docker ps | grep getting-atarted
```

以下のコマンドでコンテナを停止します。

```Bash
docker stop <container id> 
```



## 2.	Dockerfileからオリジナルコンテナを作成、DockerhubへPush


このセクションでは、Dockerfileと呼ばれるDocker Imageを作るための設定ファイルを作成して、実際にオリジナルのImageを作成します。

Imageの作成後は、DockerHubにPushを行います。


まず、以下のコマンドで作業ディレクトリに移動します。


```Bash
cd hands-on
pwd
```


続いて、image buildを行います。

docker buildコマンドではPush先のリポジトリを指定し、任意のタグをつけることができます。

その際、タグ名はユニークなものを設定します。


KubernetesのセクションでプライベートリポジトリからImageをPullするシナリオがあります。


Kubernetesのハンズオンを実施される方はプライベートリポジトリ名を指定してください。


```Bash
docker build -t <DockerHubのユーザ名>/<リポジトリ名>:<任意のタグ名> .
```

以下のコマンドで、作成したDocker Imageをコンテナアプリケーションとして起動します。


```Bash
docker run -d -p 8888:80 <DockerHubのユーザ名>/<リポジトリ名>:<任意のタグ名>
```


curlまたはブラウザを使ってアクセスすると、ご自身の作成したコンテンツが表示されるはずです。

```Bash
curl localhost:8888

```



確認後、以下のコマンドでコンテナを停止します。


```Bash
docker container ls
docker stop <container id> 
```


続いて、以下のコマンドでDocker HubにPushを行います。


```Bash
docker push <DockerHubのユーザ名>/<リポジトリ名>:<任意のタグ名>
```

## 3. Dockerfileのベストプラクティス

コンテナを作成するにあたって、Docker公式でもベストプラクティスが紹介されています。
https://docs.docker.jp/develop/develop-images/dockerfile_best-practices.html

この中で重要と思われる項目について抜粋して、簡単に補足説明します。

- 一時的なコンテナを作成
  コンテナは、可能な限り一時的（ephemeral）であるべきです。コンテナは停止と同時に破棄される一時的な環境とすることが求められます。これにより再現性を高めることに繋がります。(docker commitはNG)
  運用で必要となる永続的なデータやログなどはコンテナの外に保管するようにしましょう。

  
- .dockerignore で除外
  コンテナに不要なデータは.dockerignoreでイメージを作成する際の対象から除外することができます。特に機密情報(パスワードやAPIキーなど、認証情報)はコンテナに含まれないようにしましょう。


- マルチステージビルドを使う
  コンテナは最小のイメージとすることが望ましいです。マルチステージビルドを活用することで、ソースコードのコンパイル用ステージとコンパイル済みバイナリの実行用ステージなどを分けることができ、大幅にイメージサイズを縮小することができます。


- 不要なパッケージのインストール禁止
　脆弱性などの対象範囲を狭めることにもつながり、セキュリティが向上します。


- アプリケーションを切り離す
  各コンテナはただ１つだけの用途を持つように作成しましょう。


Docker公式では明記されていないものの、これまでのコンテナ利用実績からベストプラクティスをインターネット上で公開している企業もあります。Sysdig社のベストプラクティスは整理されていて参考としやすいため紹介します。
https://sysdig.jp/blog/dockerfile-best-practices/

その中でもいくつか重要と思われる項目について抜粋して、簡単に補足説明します。


- 不要な特権を避ける
  rootで実行すると危険です。コンテナでプログラムをroot (UID 0) として実行しないようにしましょう。DockerfileでもUSERを設定して別ユーザでプロセスを起動させるように記載します。


- 信頼できるベースイメージを使う
  信頼されていないイメージやメンテナンスされていないイメージの上にビルドすると、そのイメージの問題や脆弱性をすべてコンテナに継承してしまいます。


ハンズオンとして、マルチステージビルドを実施してみましょう。
before、afterディレクトリ配下のアプリをそれぞれbuildしてください。


```Bash
cd ..
cd multistage/before 
docker build --network host -t multistage:before .
```



```Bash
cd ..
cd after
docker build --network host -t multistage:after .
```

それぞれのサイズを比較してみましょう。1GBほど縮小されていることがわかります。

```Bash
docker images | grep multistaged
```


コンテナにアクセスしてどちらのアプリも応答することを確認してみましょう。

```Bash
docker run --rm --name echo-after -p 1323:1323 -dit multistaged:after

curl http://localhost:1323/
```

コンテナを停止して、削除されたことを確認しましょう。
```Bash
docker stop echo-after
docker ps -a
```

