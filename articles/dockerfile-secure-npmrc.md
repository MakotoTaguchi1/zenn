---
title: "Dockerでnpmrcをセキュアに扱う"
emoji: "🐳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Docker, npm, CI]
published: false
---

# 背景

Github Private Packageにnpmパッケージを社内で使用しており、それをnpmインストールする際にアクセストークンとして、.npmrcを使用していました。
Dockerでの.npmrcの取り合え使いをセキュア化した話について、共有します。

ちなみに.npmrcについて基本的な情報は、以下の記事がわかりやすかったのでご参照ください。
https://qiita.com/marumaru0113/items/21b600c21caf5d9b9775

# ビフォーアフター

## Before

```Dockerfile
## -- Build Stage
FROM node:18.20.3 AS build

WORKDIR /usr/src/app

COPY package*.json .npmrc .
RUN npm ci
COPY . .
RUN npm run build

## -- Production Stage
FROM node:18.20.3

WORKDIR /usr/src/app
...COPY系の処理など

CMD [ "node", "dist/main.js" ]
```

ただし、これはセキュアな書き方ではないので避けたいです。
理由としては、.npmrcはプライベートnpmレジストリへのアクセストークンであり秘匿情報なので、Dockerイメージに直接コピーするとイメージレイヤに保存されてしまいます。
そうなると、`docker history`コマンドなどにより、過去のレイヤーに含まれる.npmrcファイルの内容を確認できてしまう可能性があります。

## After

最終的に、Dockerfileはこうなりました。

```Dockerfile
COPY package*.json .
RUN --mount=type=secret,id=npmrc,target=.npmrc npm ci
COPY . .
RUN npm run build
```

DockerにはBuild secretというものがあり（知らなかった）、それを使って`Secret mounts`することで、ビルドの間に限り秘匿情報をビルドコンテナ内に引き渡すことができます。
つまり、Dockerレイヤに残さない形で、.npmrcの情報を利用できます。

https://docs.docker.com/build/building/secrets/#secret-mounts

【補足】
`target`がマウント先のパス・ファイル名になります。
`dst`, `destination`でも同じ意味で使えます。（[参考](https://docs.docker.jp/storage/bind-mounts.html)）

【注意点】
そのままだと、COPY . . でローカルに配置している.npmrcがレイヤに乗ってしまうので、以下をdockerignoreに追記してください。

```.dockerignore
dist
node_modules
.npmrc # 追記
```


## ビルド時の受け渡し方
このnpmrcの内容をシークレットとして利用するには、ビルド時にidを引き渡す必要があります。
例として、Docker Composeの時と、GithubActionsで利用する場合を示します。

### 注意点
DockerのBuild Secretを使うには、`BuildKit`というビルダーを使う必要があります。古いDockerエンジンではこれが無効です。

v23.0 以降の Docker Desktop, Docker Engine はデフォルトのビルダーとして BuildKit が使用されています。
それ以前のバージョンを使用する場合は明示的にBuidKitを有効化する必要があるため、注意してください。


### Docker Composeの場合

シンプルに、この形でシークレットを引き渡せます。
1. ローカルのリポジトリルートに.npmrcファイルを配置。
2. トップレベルのSecretディレクティブでローカルにある.npmrcファイルを指定。
3. マウントしたコンテナにて、secrets attributeで指定。
4. 環境変数で`DOCKER_BUILDKIT: 1`を指定して、明示的に有効化（dockerバージョンは開発者環境によるため）

docker-compose.yaml
```yaml
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      secrets:
        - npmrc    # シークレットの指定
    environment:
      DOCKER_BUILDKIT: 1

secrets:
  npmrc:
    file: .npmrc  # ローカルの.npmrcファイルのパス
```


### Github Actionsの場合

1. リポジトリシークレットにNPM_TOKENとして保存しておき.npmrcファイルを動的に生成
2. docker buildコマンドのsecretオプションにて指定

```yaml
      - name: docker build & push
        run: |
          echo "engine-strict=true" >> ./.npmrc 
          echo "//registry.npmjs.org/:_authToken=${{ secrets.NPM_TOKEN }}" >> ./.npmrc
          docker --version
          docker build --secret id=npmrc,src=.npmrc --pull -t ${GAR_REPO}:${{ env.COMMIT_SHA }} .
          docker push ${GAR_REPO}:${{ env.COMMIT_SHA }}
```

【補足】
GihubActionsのubuntu-latestランナーでは、デフォルトでDockerがインストールされています。
実際にワークフロー実行して確認したところ、Dockerバージョンは26.1.3でした。（2024年12月現在）
v23より新しいので、BuildKitはデフォルトで有効になっており明示的な有効化不要です。
気になるようでしたら、念のため`docker --version`で確認しておくと良いと思います。


# 参考

- https://zenn.dev/tksx1227/articles/4af1ce9b9e475a
- https://qiita.com/taquaki-satwo/items/f8fbe8b1efc4b2323ae7#7-%E3%82%B9%E3%83%86%E3%83%83%E3%83%97%E3%81%AE%E9%A0%86%E7%95%AA%E3%82%92%E6%9C%80%E9%81%A9%E5%8C%96%E3%81%99%E3%82%8B


