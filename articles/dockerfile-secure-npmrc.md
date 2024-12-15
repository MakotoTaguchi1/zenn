---
title: "Dockerfileで.npmrcをセキュアに扱う【Build secrets】"
emoji: "🐳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Docker, npm, CI]
published: true
---

# 背景

社内で Github Package でプライベート npm パッケージを運用しており、それを npm インストールするためのアクセストークンとして`.npmrc`を使用しています。
最近、Dockerfile での.npmrc の取り扱いをセキュア化しました。その過程で少しだけコツと注意がいったので、共有したいと思います。

ちなみに.npmrc とはなんぞ？ということ本記事では省略します。ご存知なければ以下の記事が分かりやすかったのでご参照ください。
https://qiita.com/marumaru0113/items/21b600c21caf5d9b9775

# ビフォーアフター

## Before

ルートディレクトリに.npmrc を配置しておき、pakage.json, package-lock.json と共にコピーして npm ci していました。
ちなみに本題とはそれますが、Dockerfile のベストプラクティスに則り、いきなり`COPY . .`するよりも、この方が Docker のキャッシュが効きやすいため順番を最適化しています。

```Dockerfile
## -- Build Stage
FROM node:18.20.3 AS build
WORKDIR /usr/src/app

# 本題はここ↓
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

**ただし、これはセキュアな書き方ではないので避けたいです。**
理由は、.npmrc はプライベート npm レジストリへのアクセストークンであり秘匿情報なので、**Docker イメージに直接コピーするとイメージレイヤに保存されてしまうからです。**
そうなると、`docker history`コマンドなどにより、過去のレイヤーに含まれる.npmrc ファイルの内容を確認できてしまうことになります。

## After

抜粋ですが最終的に Dockerfile はこうなりました。
.npmrc は COPY するのではなく、一時的なシークレット情報として引き渡す形になりました。

```Dockerfile
COPY package*.json .
RUN --mount=type=secret,id=npmrc,target=.npmrc npm ci
COPY . .
RUN npm run build
```

Docker には `Build Secrets` という機能があり、その機能の一部である`Secret mounts`を利用することで、**ビルドの間に限り秘匿情報をビルドコンテナ内に引き渡す**ことができます。
**つまり、Docker レイヤに.npmrc の情報を残さない形で利用できます。**

https://docs.docker.com/build/building/secrets/#secret-mounts

【補足】
`target`がマウント先のパス・ファイル名になります。
`dst`, `destination`でも同じ意味で使えます。（[参考](https://docs.docker.jp/storage/bind-mounts.html)）

### 【重要: 注意点】

そのままだと、`COPY . .` でローカルに配置している.npmrc が結局 Docker レイヤに乗ってしまいます。
**`.dockerignore`に追記して COPY されないようにしてください。**

```.dockerignore
dist
node_modules
.npmrc # 追記
```

# ビルド時のシークレットの受け渡し方

上記により、ビルド時に.npmrc をシークレットとしてセキュアに引き渡せるようになりました。
次は、実際のビルド時の secret id の引き渡し方です。

例として、**Docker Compose**と、**Github Actions**で利用する場合をそれぞれ示します。

:::message
Docker の Build Secret を使うには、`BuildKit`というビルダーを使う必要があります。
古い Docker エンジンではこれが無効です。
v23.0 以降の Docker Desktop, Docker Engine はデフォルトのビルダーとして BuildKit が使用されています。
それ以前のバージョンを使用する場合は明示的に BuidKit を有効化する必要があります。環境によっては注意してください。
:::

## Docker Compose の場合

以下の手順です。

1. ローカルのリポジトリルートに.npmrc ファイルを配置。
2. トップレベルの `secrets` ディレクティブでローカルにある.npmrc ファイルの所在地を指定。
3. Dockerfile を利用するコンテナの `secrets` にて、上記 secret を指定。
4. 環境変数で`DOCKER_BUILDKIT: 1`を指定して、明示的に有効化すると良い（docker バージョンは開発者環境によるため）

docker-compose.yaml（抜粋）

```yaml
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      secrets:
        - npmrc # 3.
    environment:
      DOCKER_BUILDKIT: 1 # 4.

secrets:
  npmrc:
    file: .npmrc # 2.
```

## Github Actions の場合 ß

以下の手順です。

1. リポジトリシークレットに.npmrc のトークンを`NPM_TOKEN`として保存しておく。
2. .npmrc ファイルを動的に生成。
3. docker build コマンドの secret オプションにて指定。`src`にて先で作った.npmrc のパスを指定できる。

```yaml
- name: docker build & push
  run: |
    echo "engine-strict=true" >> ./.npmrcß
    echo "//registry.npmjs.org/:_authToken=${{ secrets.NPM_TOKEN }}" >> ./.npmrc
    docker --version
    docker build --secret id=npmrc,src=.npmrc --pull -t ${GAR_REPO}:${{ env.COMMIT_SHA }} .
    docker push ${GAR_REPO}:${{ env.COMMIT_SHA }}
```

【補足】
GihubActions の ubuntu-latest ランナーでは、デフォルトで Docker がインストールされています。
実際にワークフロー実行して確認したところ、Docker バージョンは 26.1.3 でした。（2024 年 12 月現在）
v23.0 より新しいので、BuildKit はデフォルトで有効になっています。そのためここでは明示的に有効化しませんでした。
一応、ワークフローに`docker --version`を書いておくと良いです。

# 参考

https://zenn.dev/tksx1227/articles/4af1ce9b9e475a

https://qiita.com/taquaki-satwo/items/f8fbe8b1efc4b2323ae7#7-%E3%82%B9%E3%83%86%E3%83%83%E3%83%97%E3%81%AE%E9%A0%86%E7%95%AA%E3%82%92%E6%9C%80%E9%81%A9%E5%8C%96%E3%81%99%E3%82%8B
