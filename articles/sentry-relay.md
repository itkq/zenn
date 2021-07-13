---
title: "Sentry Relay を活用する"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["sentry"]
published: true
---

エラートラッキングサービスとしての Sentry は知名度も高く、Web サービスを運用する上で活用している人が多いでしょう。本記事では Sentry 公式が開発している [Relay](https://docs.sentry.io/product/relay/) というプロダクトについて紹介します。

# Relay の概要
Relay は、クライアントと Sentry サーバの中間に配置する、リレーサーバです。エラートラッキングとして Sentry を利用する際、エラーを投げる先である [DSN](https://docs.sentry.io/product/sentry-basics/dsn-explainer/) を設定します。Relay を挟む際は、DSN のホストが Relay のそれになります。https://github.com/getsentry/relay にて Apache License 2.0 で公開されています。

# 4 つのユースケース

1 〜 3 は公式ドキュメントにあるもので、4 は個人的に使えそうなものを挙げています。

## 1. 個人情報フィルタリング

特に SaaS である sentry.io を使う場合、エラー情報にメールアドレスなどの個人情報が載らないように注意する必要があります。個人情報をフィルタする方法は 3 つあります。

1. [SDK](https://docs.sentry.io/platforms/javascript/data-management/sensitive-data/)
2. [Server-side](https://docs.sentry.io/product/data-management-settings/scrubbing/server-side-scrubbing/)
3. Relay

SDK では `before-send` hook を使い、Sentry サーバへ送信する前に情報を落とせます。JavaScript コード例は以下です (公式ドキュメントからそのまま引っ張ってきたものです)。**アプリケーション単位で設定する必要があります**。

```javascript
Sentry.init({
  dsn: "...",
  beforeSend(event) {
    // Modify the event here
    if (event.user) {
      // Don't send user's email address
      delete event.user.email;
    }
    return event;
  },
});
```

次に Server-side とは、sentry.io 上で個人情報をフィルタするものです。**個人情報自体はサーバに送信されますが、それをストアしないようにできます**。このフィルタ設定はプロジェクト単位です。Data Scrubber を有効にすればデフォルトで password などの値がフィルタされます。加えて独自の設定も可能です。

本題である Relay を活用すると、**フィルタリングの設定を中央管理しつつ、Sentry サーバに個人情報を送信しなくなります**。Relay の動作モードのうち managed を選択すると 、Server-side の設定を読み、Relay 上でフィルタリングを行ってくれます。コンプライアンスやセキュリティ要件によっては Relay が良いソリューションになるでしょう。

## 2. レスポンスタイムの削減
Relay はレスポンスを素早く返すように設計されているため、直接 Sentry サーバにイベントを送信するよりレスポンスが高速になるようです。これによりイベント送信が頻繁に行われるアプリケーションのパフォーマンスを改善できる可能性があります。

## 3. 制限されたドメインでの使用
使用できるアウトバウンドのドメインに制限があり、sentry.io を直接利用できない環境では、何らかのプロキシを使用することになります。このプロキシとして Relay を利用できます。

## 4. 統計情報の監視
Relay は [StatsD 形式で様々なメトリクスを出力します](https://docs.sentry.io/product/relay/monitoring/collected-metrics/)。例えば event accepted, dropped 数などがあります。sentry.io を運用する上で気をつけなければならないことの 1 つが、月間の event limit 超過により event が drop されてしまうことです。Subscription のページから、現在の event usage は閲覧できますが、利用割合などを入力に通知はできません。そこで、Relay のメトリクスを使って、監視やアラーティングとして利用できます。

https://github.com/prometheus/statsd_exporter をサイドカーとしてデプロイし、メトリクス送信先の設定をすれば Prometheus にメトリクスを吸わせることもできます。Grafana と組み合わせると、例えば以下のように event accepted 数を可視化できます。
![](https://storage.googleapis.com/zenn-user-upload/35088ab4c9f4902e0cb792c9.png)

# Pros/Cons

これまでユースケースの説明をしてきましたが、実際に導入するにあたって Pros/Cons をまとめます。

## Pros

### Officially Supported

公式にメンテナンスされているため、自前でリレーサーバを書くよりも機能拡張や堅牢性の観点で良いです。

### Container ready

公式に [Docker イメージ](https://hub.docker.com/r/getsentry/relay/)が提供されています。[Operating Guidelines](https://docs.sentry.io/product/relay/operating-guidelines/) でもコンテナを念頭に置いた記述がされています。K8s などへ手軽にデプロイできます。

### 他のコンポーネントが不要

ストレージなど他のコンポーネントは不要なため、運用は比較的しやすいと思われます。

## Cons

### 運用負荷の増加

当然のことではありますが、sentry.io というマネージドなサービスを使っていたとしても、Relay は自前で運用する必要があります。運用しやすいとはいえ、Relay 自体の監視やアラーティングを作り込んだほうが良いでしょう。運用にあたっては [Operating Guidelines](https://docs.sentry.io/product/relay/operating-guidelines/) も参考になります。


# K8s へのデプロイ
最後に、K8s を対象とした具体的なデプロイについて簡単に書きます。

[Configuration Options](https://docs.sentry.io/product/relay/options/) の通り、設定はファイルを用意して引数指定により行います。ソースコードを読むと、一部環境変数による指定が可能ですが、すべてではないため、ファイルの用意が無難です。ConfigMap を次のように定義します。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: sentry-relay-config
data:
  config.yml: |
    relay:
      mode: proxy
      upstream: "https://sentry.io/"
      host: 0.0.0.0
      port: 3000
      tls_port: ~
      tls_identity_path: ~
      tls_identity_password: ~
    metrics:
      statsd: 127.0.0.1:9125
      prefix: sentry.relay
    logging:
      format: json
```

Deployment では config.yml をマウントしてあげれば良いです。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    name: sentry-relay
  name: sentry-relay
spec:
  replicas: 2
  selector:
    matchLabels:
      name: sentry-relay
  template:
    metadata:
      labels:
        name: sentry-relay
    spec:
      containers:
      - name: app
        image: getsentry/relay:21.5.1
        ports:
        - name: sentry-web
          containerPort: 3000

        # https://docs.sentry.io/product/relay/monitoring/
        readinessProbe:
          httpGet:
            path: /api/relay/healthcheck/live/
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /api/relay/healthcheck/ready/
            port: 3000
          initialDelaySeconds: 20
          periodSeconds: 10

        resources:
          requests:
            cpu: 1000m
            memory: 512M
          limits:
            cpu: 1000m
            memory: 512M

        volumeMounts:
        - name: sentry-relay-config-volume
          mountPath: /work/.relay/config.yml
          subPath: config.yml

      - name: statsd-exporter
        image: prom/statsd-exporter:v0.20.2
        args: ["--log.format=json"]
        ports:
        - name: statsd-web
          containerPort: 9102
        - name: statsd-metrics
          containerPort: 9125

      volumes:
      - name: sentry-relay-config-volume
        configMap:
          name: sentry-relay-config

```

あとは Service を足すなどしてルーティングしてあげれば使用可能です。

# まとめ

いくつかのユースケースがある Sentry Relay というプロダクトを紹介しました。自分が所属している Ubie では Relay を導入して運用してみていますが、元気に働いてくれています。
