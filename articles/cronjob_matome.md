---
title: "CronJob使い始めてハマった箇所まとめてみた"
emoji: "📚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [k8s, gke, cronjob]
published: false
---

# はじめに

# Environment

筆者が今回使用した環境は以下です。

- GKE Autopilot リージョンクラスター
- K8s Version: 1.29.7

# ハマりポイント

## schedule のタイムゾーン

基本的に、デフォルトだと UTC タイムゾーンになる環境が多いようです。（GKE でもそうでした。）
そのため、日本時間では +9 時間でセットする必要があります。

```yaml
spec:
  # これは,日本時間 毎日 19時 に実行される
  schedule: "0 10 * * *"
  jobTemplate:
    # ...ジョブの記述
```

しかしながら Ks8 v1.27 以降では、`.spec.timeZone`がサポートされ、ユーザ側でタイムゾーンを設定できるようになったようです。

```yaml
spec:
  timeZone: "Asia/Tokyo"
  schedule: "0 10 * * *"
  jobTemplate:
    # ...ジョブの記述
```

> You can specify a time zone for a CronJob by setting .spec.timeZone to the name of a valid time zone. For example, setting .spec.timeZone: "Etc/UTC" instructs Kubernetes to interpret the schedule relative to Coordinated Universal Time.

https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/#time-zones

## サイドカーコンテナを立てたいとき

Pod 中に複数のコンテナを立てるサイドカー構成にしたいときがあります。
自分も今回、GKE Pod から CloudSQL インスタンスにアクセスしたかったので`cloud-sql-proxy`を使いたかったです。

この時は注意が必要です。

deployment と同じように、`template.spec.containers`にメインコンテナと並列にサイドカーコンテナを記述すると、
メインコンテナのジョブが終了（Completed）してもサイドカーが起動し続ける（Running）ため、永久に CronJob が終了しない状況に陥ります。
（自分もハマりました）

そこで、K8s v1.29 より beta に昇格した `SidecarContainers`という機能が便利です。

この機能では、`initContainers`というブロックにサイドカーコンテナを書き、`restartPolicy: Always`とするとメインコンテナ終了後もジョブの完了を妨げないようになります。

```yaml

```

https://kubernetes.io/ja/docs/concepts/workloads/pods/sidecar-containers/

## GKE Autopilot のリソース配置

resource request / limit についてです。
GKE Autopilot にデプロイできる Pod について request / limit の制約があります。
配置可能な最小/最大リクエストや cpu/memory の比率、バースト可否など、いくつかの制約があります。

詳しくは、以下ご覧ください。

https://cloud.google.com/kubernetes-engine/docs/concepts/autopilot-resource-requests?hl=ja

2024 年 8 月現在、CronJob では、以下を気にする必要があるようです。

- request, limit は両方明記する必要あり。（Deployment 等では省略してもデプロイ可能。Autopilot 側でデフォルト値が付与される）
- request = limit 値とする必要あり。（おそらくバースト不可）

kubectl diff すると以下のようなエラーが出ました。apply も失敗します。

```
# request, limit書かない時
Violations details: {"[denied by autogke-pod-limit-constraints]":["container '<コンテナ名>' does not have resources/limits defined for all resources which is required in Autopilot clusters."]}

# request = limit値としなかった時
Violations details: {"[denied by autogke-pod-limit-constraints]":["container '<コンテナ名>' does not have resource==limits which is required in Autopilot clusters."]}
```

特にバーストできないのは微妙ですが、Autopilot 側の改善を待ちたいです。

## restartPolicy, backoffLimit

GKE に限らず CronJob を扱う上で気を付けるべき箇所です。
失敗時の再実行については、オプションを把握して検討しておく必要があります。

### restartPolicy

`Never`または`OnFailure`が指定できます。

#### Never

- ジョブが失敗すると、その Pod は再実行されない。
- ただし、**backoffLimit に達するまで新しい Pod でジョブが再実行される。（誤解しやすい）**
  - そのため異なるノードで実行される可能性あることや状態は保持されないことは留意必要。
- 失敗した Pod は残る
  - そのため後からログを確認できる。

#### OnFailure

- ジョブが失敗すると、同じ Pod 内でコンテナが再起動される。
- backoffLimit に達するまで同じ Pod でジョブが再実行される。
- リトライ上限に達すると、失敗した Pod は削除される
  - そのため後からログが確認できない可能性あり。

### backoffLimit

ジョブ失敗時に何回まで再起動するかを設定するオプションです。

- デフォルト値は 6 で、省略するとこれが適用される
- リトライ間隔は指数関数的に増加する （exponential backoff）

再実行させたくない場合は、restartPolicy を Never に、backoffLimit を 0 に設定すると良さそうです。

# さいごに

# 参考

- https://medium.com/devopsturkiye/how-to-set-timezone-for-kubernetes-cronjobs-691d3aaa34ef
- https://zenn.dev/cloud_ace/articles/b5b16938a87770
