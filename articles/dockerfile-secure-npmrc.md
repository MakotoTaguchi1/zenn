---
title: "Dockerfileで.npmrcをセキュアに扱う【Build secrets】"
emoji: "🐳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Docker, npm, CI]
published: true
---

# 背景

社内で Github Package でプライベートnpmパッケージを運用しており、それをnpmインストールするためのアクセストークンとして`.npmrc`を使用しています。
最近、Dockerfileでの.npmrcの取り扱いをセキュア化しました。その過程で少しだけコツと注意がいったので、共有したいと思います。

ちなみに.npmrcとはなんぞ？ということ本記事では省略します。ご存知なければ以下の記事が分かりやすかったのでご参照ください。
https://qiita.com/marumaru0113/items/21b600c21caf5d9b9775

# ビフォーアフター

## Before

ルートディレクトリに.npmrcを配置しておき、pakage.json, package-lock.jsonと共にコピーしてnpm ciしていました。
ちなみに本題とはそれますが、Dockerfileのベストプラクティスに則り、いきなり`COPY . .`するよりも、この方がDockerのキャッシュが効きやすいため順番を最適化しています。

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
理由は、.npmrcはプライベートnpmレジストリへのアクセストークンであり秘匿情報なので、**Dockerイメージに直接コピーするとイメージレイヤに保存されてしまうからです。**
そうなると、`docker history`コマンドなどにより、過去のレイヤーに含まれる.npmrcファイルの内容を確認できてしまうことになります。

## After

抜粋ですが最終的にDockerfileはこうなりました。
.npmrcはCOPYするのではなく、一時的なシークレット情報として引き渡す形になりました。

```Dockerfile
COPY package*.json .
RUN --mount=type=secret,id=npmrc,target=.npmrc npm ci
COPY . .
RUN npm run build
```

Dockerには `Build Secrets` という機能があり、その機能の一部である`Secret mounts`を利用することで、**ビルドの間に限り秘匿情報をビルドコンテナ内に引き渡す**ことができます。
**つまり、Dockerレイヤに.npmrcの情報を残さない形で利用できます。**

https://docs.docker.com/build/building/secrets/#secret-mounts

【補足】
`target`がマウント先のパス・ファイル名になります。
`dst`, `destination`でも同じ意味で使えます。（[参考](https://docs.docker.jp/storage/bind-mounts.html)）

### 【重要: 注意点】
そのままだと、`COPY . .` でローカルに配置している.npmrcが結局Dockerレイヤに乗ってしまいます。
**`.dockerignore`に追記してCOPYされないようにしてください。**

```.dockerignore
dist
node_modules
.npmrc # 追記
```


# ビルド時のシークレットの受け渡し方
上記により、ビルド時に.npmrcをシークレットとしてセキュアに引き渡せるようになりました。
次は、実際のビルド時のsecret idの引き渡し方です。

例として、**Docker Compose**と、**Github Actions**で利用する場合をそれぞれ示します。

:::message
DockerのBuild Secretを使うには、`BuildKit`というビルダーを使う必要があります。
古いDockerエンジンではこれが無効です。
v23.0 以降の Docker Desktop, Docker Engine はデフォルトのビルダーとして BuildKit が使用されています。
それ以前のバージョンを使用する場合は明示的にBuidKitを有効化する必要があります。環境によっては注意してください。
:::

## Docker Composeの場合

以下の手順です。
1. ローカルのリポジトリルートに.npmrcファイルを配置。
2. トップレベルの `secrets` ディレクティブでローカルにある.npmrcファイルの所在地を指定。
3. Dockerfileを利用するコンテナの `secrets` にて、上記secretを指定。
4. 環境変数で`DOCKER_BUILDKIT: 1`を指定して、明示的に有効化すると良い（dockerバージョンは開発者環境によるため）

docker-compose.yaml（抜粋）
```yaml
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      secrets:
        - npmrc  # 3.
    environment:
      DOCKER_BUILDKIT: 1  # 4.

secrets:
  npmrc:
    file: .npmrc  # 2.
```


## Github Actionsの場合ß

以下の手順です。
1. リポジトリシークレットに.npmrcのトークンを`NPM_TOKEN`として保存しておく。
2. .npmrcファイルを動的に生成。
3. docker buildコマンドのsecretオプションにて指定。`src`にて先で作った.npmrcのパスを指定できる。

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
GihubActionsのubuntu-latestランナーでは、デフォルトでDockerがインストールされています。
実際にワークフロー実行して確認したところ、Dockerバージョンは 26.1.3 でした。（2024年12月現在）
v23.0より新しいので、BuildKitはデフォルトで有効になっています。そのためここでは明示的に有効化しませんでした。
一応、ワークフローに`docker --version`を書いておくと良いです。


# 参考

https://zenn.dev/tksx1227/articles/4af1ce9b9e475a

https://qiita.com/taquaki-satwo/items/f8fbe8b1efc4b2323ae7#7-%E3%82%B9%E3%83%86%E3%83%83%E3%83%97%E3%81%AE%E9%A0%86%E7%95%AA%E3%82%92%E6%9C%80%E9%81%A9%E5%8C%96%E3%81%99%E3%82%8B


