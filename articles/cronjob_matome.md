---
title: "CronJobのハマりポイントまとめてみた"
emoji: "⏰"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [k8s, gke, cronjob]
published: true
---

# はじめに

最近業務で Kubernetes（K8s） CronJob を使った定期実行ジョブを構築する機会がありました。
一から構築するのは初めてだったこともあり、初歩的なことから地味に見事ハマりまくりました。

苦戦しつつも一旦形にはできたので、同じ境遇の方の役に立つかもと思い、ハマりポイントを記録してみました。

# Environment

筆者が今回使用した環境は以下の通りです。

- GKE Autopilot リージョンクラスター
- K8s Version: `1.29.7`

# CronJob とは

今回のメインではないので、簡単な説明に留めます。

K8s の CronJob は、K8s 環境でコンテナベースのタスクにより定期的にジョブ（Job）を実行するためのリソースです。用途としては、定期的なデータ処理・リソースのクリーンアップ・定期通知やレポート送信などで使われるケースがあると思います。

クラウドマネージドサービス（例: Google Cloud Scheduler + 旧 Cloud Functions）でも定期ジョブは実現できます。

一方 CronJob のメリットは、例えば以下が挙げられると思います。

- K8s の成長に相乗りできる
- ランタイムの制約を受けない
- オンプレミスや複数クラウドのハイブリッド環境でも一貫したジョブ管理をできる

# ハマったポイント

## 1. schedule のタイムゾーン

### 注意点

CronJob では`schedule`というオプションでそのジョブをいつ起動させるかを設定します。
UNIX 系システムで使用される Cron 表現形式で書きます。

**ところがデフォルトだと UTC タイムゾーンになる環境が多いようです。GKE でも UTC でした。**
そのため日本時間は +9 時間して schedule にセットする必要があります。

```yaml
spec:
  # これはUTCなので、日本時間 毎日 19時 に実行される
  schedule: "0 10 * * *"
  jobTemplate:
    # ...ジョブの記述
```

### 解決策

K8s v1.27 以降では、`.spec.timeZone`がサポートされ、**ユーザ側でタイムゾーン自由に設定できるようになりました。**

```yaml
spec:
  # これはSTなので、毎日 10時 に実行される
  timeZone: "Asia/Tokyo"
  schedule: "0 10 * * *"
  jobTemplate:
    # ...ジョブの記述
```

これにより、時刻セットミスのない運用ができるようになりました。

> You can specify a time zone for a CronJob by setting .spec.timeZone to the name of a valid time zone. For example, setting .spec.timeZone: "Etc/UTC" instructs Kubernetes to interpret the schedule relative to Coordinated Universal Time.

https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/#time-zones

## 2. サイドカーコンテナを立てるとき

### 注意点

Pod 中に複数のコンテナを立てるような、サイドカー構成にしたいときがあります。
自分も今回、GKE Pod から CloudSQL インスタンスにアクセスしたかったので、メインコンテナと別に`cloud-sql-proxy`をサイドカーを記述しました。

この時、注意が必要です。

deployment と同じように`template.spec.containers`で、メインコンテナと並列にサイドカーコンテナを記述すると、
**メインコンテナのジョブが正常終了（Completed）しても、サイドカーが起動し続ける（Running）ため、ジョブが終了せず、永久に Pod が起動し続ける状況に陥ります。**

これは、すべてのコンテナが正常終了（Completed 状態）になるまで、その Pod が完了したとみなされないという Job の仕様によるものです。

### 解決策

**これを解決するためには、K8s v1.29 より beta に昇格した `SidecarContainers`という機能が便利です。**

この機能では、

- `initContainers`というブロックにサイドカーコンテナを記述し、（`containers`ではない）
- `restartPolicy: Always`とすることで

メインコンテナ終了後にジョブ完了し、サイドカーが完了を妨げないようになります。

↓ 記述例（抜粋）

```yaml
template:
  spec:
    # メインコンテナ
    containers:
      - name: main-container
        image: asia.gcr.io/hoge-project/hoge-repo:tag
        command: ["/bin/sh"]
        args:
          - "-c"
          - " npm run command"

    # サイドカーコンテナ
    initContainers:
      - name: cloud-sql-proxy
        image: gcr.io/cloud-sql-connectors/cloud-sql-proxy:2.13.0
        args:
          - "--structured-logs"
          - "--port=3306"
          - "<CloudSQLインスタンス接続名>"
        restartPolicy: Always # ここも記述いる
        securityContext:
          runAsNonRoot: true
```

https://kubernetes.io/ja/docs/concepts/workloads/pods/sidecar-containers/

## 3. restartPolicy と backoffLimit

ジョブ失敗時の挙動に関するオプションで、GKE に限らず CronJob を扱う上で気を付けるべき箇所です。
関連するオプションの特性を把握して、リトライ処理を検討する必要があります。

### 3.1 restartPolicy

CronJob では、`Never`または`OnFailure`が指定できます。
以下にポイントを列挙します。

#### Never

- ジョブが失敗すると、その Pod は再実行されない。
- **【ここ誤解しやすい】 同じ Pod では再実行されないが、backoffLimit（後述） に達するまで新しい Pod でジョブが再実行される。**
  - 異なるノードで実行される可能性あることや、状態は保持されないことは留意必要。
- 失敗した Pod は残る。
  - そのため後からログを確認しやすい。

#### OnFailure

- ジョブが失敗すると、同じ Pod 内でコンテナが再起動される。
- backoffLimit に達するまで同じ Pod でジョブが再実行される。
- リトライ上限に達すると、失敗した Pod は削除される
  - そのため後からログが確認できない可能性あり。

### 3.2 backoffLimit

ジョブ失敗時に何回まで再起動するかを設定するオプションです。

- デフォルト値は 6 で、省略するとこれが適用される
- リトライ間隔は指数関数的に増加する （exponential backoff）

**再実行させたくない場合は、restartPolicy を `Never` に、backoffLimit を 0 にすると良さそうです。**

## 4. GKE Autopilot のリソース配置

この節は GKE Autopilot 限定なので、別環境で検討されている方は読み飛ばしてください。

### 【背景】Autopilot のリソース制約について

resource request, limit についてです。
GKE Autopilot にデプロイする Pod は 配置できる cpu / memory の request, limit の値に制約があります。
配置可能な最小/最大リクエストや cpu/memory の比率、バースト可否など、いくつかの制約があります。

詳しくは、以下ご覧ください。

https://cloud.google.com/kubernetes-engine/docs/concepts/autopilot-resource-requests?hl=ja

### 注意点

2024 年 8 月現在、GKE Autopilot で CronJob でを使う場合、以下 2 点を気にする必要があるようです。

1. `request`, `limit` は両方明記する必要あり。（Deployment 等では省略してもデプロイ可能。Autopilot 側でデフォルト値が付与される）
2. `request = limit` 値とする必要あり。（おそらくバースト不可）

実際に、これらに違反した書き方をして kubectl diff すると以下のようなエラーが出ました。apply も失敗します。

```
# request, limit書かない時
Violations details: {"[denied by autogke-pod-limit-constraints]":["container '<コンテナ名>' does not have resources/limits defined for all resources which is required in Autopilot clusters."]}

# request = limit値としなかった時
Violations details: {"[denied by autogke-pod-limit-constraints]":["container '<コンテナ名>' does not have resource==limits which is required in Autopilot clusters."]}
```

特にバーストできないのは微妙ですが、Autopilot 側の改善を待ちたいです。

# さいごに

筆者のこれまでの経験的に、実稼働環境で CronJob を採用するケースは比較的少ないです。
（バッチはアプリケーションが稼働している K8s 環境とリソース分離したかったり、クラウドのマネージドサービスを使った方が運用が楽だったりする）

業界全体的にもその傾向があるためか、CronJob について初心者向けに日本語でまとまっている記事は多くない気がします。

自分と同じように CronJob を触り始めの方などのお役に立てれば幸いです。

# 参考

- https://medium.com/devopsturkiye/how-to-set-timezone-for-kubernetes-cronjobs-691d3aaa34ef
- https://zenn.dev/cloud_ace/articles/b5b16938a87770
