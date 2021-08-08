---
title: "graphql-spring-boot と Micrometer で始める GraphQL モニタリング"
emoji: "⚖️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["graphql", "springboot"]
published: true
---

graphql-spring-boot で実装した GraphQL サーバで、Micrometer を使って GraphQL オペレーションレベルのメトリクスを簡単に実装し、Prometheus と Grafana で可視化したという話です。

# graphql-spring-boot

Spring Boot アプリケーションを GraphQL サーバにするためのライブラリです。今回はこれを使っている前提です。
https://github.com/graphql-java-kickstart/graphql-spring-boot


# モニタリング観点での GraphQL
GraphQL over HTTP では、レスポンスの HTTP ステータスコードを 200 で固定し、レスポンスボディの `errors` フィールドの有無でエラーを表現することが多いです。そのため、普通の REST アプリケーションとは異なり、ロードバランサやプロキシのレイヤで HTTP ステータスコードを見るだけではアプリケーションレイヤーのエラーを観測できません。エラーは [Four Golden Signals](https://sre.google/sre-book/monitoring-distributed-systems/#xref_monitoring_golden-signals) の 1 つであり、アプリケーションの運用において特に観測したいメトリクスです。

また、オペレーションレベルや、フィールド・変数レベルでのメトリクスを収集できると、可観測性が向上するため嬉しくなります。

# Micrometer
[Micrometer](https://micrometer.io/) は JVM アプリケーション上でメトリクス計測を実装するためのライブラリです。Spring Boot には [Actuator](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html) という運用向けの機能があり、このうちメトリクスの実装に Micrometer が使われています[^1]。Micrometer は Registry というインターフェースを提供しており、特定の Registry を使うことでメトリクスのエクスポート先を簡単に指定できます。今回は、メトリクスを格納するモニタリングシステムに Prometheus を選び、[micrometer-registry-prometheus](https://micrometer.io/docs/registry/prometheus) を使って Prometheus 形式でメトリクスを吐けるようにします。

# メトリクスの設計と実装

筆者は GraphQL の運用経験に乏しいので、運用の一般的な知識を使いスモールスタートする方針としました。結論として、オペレーションごとの総数 (success or error)、レイテンシを出力させることにしました。エラーとレイテンシは SLI として選びやすいからというのが第一の理由です。また、REST アプリケーションにおいてパスレベルでメトリクスを出力させると、正規化が面倒という問題がある一方、GraphQL のオペレーションレベルならばそこに問題がなく利点が大きいと判断したからです。メトリクスやラベルが多いほど可観測性は向上するはずですが、アプリケーションのパフォーマンスとモニタリングシステムのキャパシティを考慮し、きちんと理解している範囲でコントローラブルにすることを念頭に置きました。

実装は、同じく Spring の GraphQL framework である dgs-framework を参考にしました。これは Netflix が開発運用している実績があり、かつ[トレーシングやメトリクスの計測](https://netflix.github.io/dgs/advanced/instrumentation/)がよくできていそうでした。

https://github.com/Netflix/dgs-framework

dgs-framework は graphql-spring-boot と同じく [GraphQL Java](https://www.graphql-java.com/) を使っています。GraphQL Java が提供する `instrumentation` を利用すればメトリクスを実装できます。今回は GraphQL Java の提供する [TracingInstrumentation](https://github.com/graphql-java/graphql-java/blob/master/src/main/java/graphql/execution/instrumentation/tracing/TracingInstrumentation.java) を継承して実装してみます。以下は Kotlin のコードです。

```kotlin
import graphql.ExecutionResult
import graphql.execution.instrumentation.parameters.InstrumentationExecutionParameters
import graphql.execution.instrumentation.tracing.TracingInstrumentation
import io.micrometer.core.instrument.MeterRegistry
import io.micrometer.core.instrument.Tag
import org.springframework.stereotype.Component
import org.springframework.util.CollectionUtils
import java.util.concurrent.CompletableFuture

@Component
class GraphQLInstrumentation(
    val meterRegistry: MeterRegistry
) : TracingInstrumentation() {
    companion object {
        const val QUERY_COUNTER_METRIC_NAME = "graphql.query"

        const val TAG_KEY_OUTCOME = "outcome"
        const val TAG_VALUE_OUTCOME_SUCCESS = "success"
        const val TAG_VALUE_OUTCOME_ERROR = "error"

        const val TAG_KEY_OPERATION = "operation"
        const val TAG_KEY_OPERATION_NAME = "operation_name"

        const val TAG_VALUE_UNKNOWN = "unknown"
    }

    override fun instrumentExecutionResult(
        executionResult: ExecutionResult?,
        parameters: InstrumentationExecutionParameters?
    ): CompletableFuture<ExecutionResult> {
        val outcome = if (CollectionUtils.isEmpty(executionResult?.errors)) {
            TAG_VALUE_OUTCOME_SUCCESS
        } else {
            TAG_VALUE_OUTCOME_ERROR
        }

        val operation = parameters?.query?.toString()?.let {
            Regex("""^(query|mutation)""").find(it)?.value ?: TAG_VALUE_UNKNOWN
        } ?: TAG_VALUE_UNKNOWN
        val operationName = parameters?.operation ?: TAG_VALUE_UNKNOWN
        val tags = listOf(Tag.of(TAG_KEY_OUTCOME, outcome), Tag.of(TAG_KEY_OPERATION, operation), Tag.of(
            TAG_KEY_OPERATION_NAME, operationName))

        meterRegistry.counter(QUERY_COUNTER_METRIC_NAME, tags).increment()

        return super.instrumentExecutionResult(executionResult, parameters)
    }
}
```

`Instrumentation` インターフェースを実装し、`@Component` アノテーションを付与するだけです。こういう体験をすると Spring 便利だなぁと思いますね。この実装では、query か mutation かを判別しつつ、オペレーション名と成功可否をタグに入れてカウンター `graphql_query_total` をインクリメントしているだけです。

続いて Spring の設定を以下のように足します。

```yaml
graphql:
  servlet:
    actuator-metrics: true
    tracing-enabled: false
management:
  endpoints:
    enabled-by-default: true
    web:
      exposure:
        include: prometheus
```

サーバを立ち上げ、GraphQL クエリを投げたのち、curl で確認すると `graphql_query_total` と `graphql_timer_query_seconds` が出力されていることを確認できます。

```
$ curl -Ssf localhost:8080/actuator/prometheus | grep graphql | grep foo
graphql_query_total{operation="query",operation_name="foo",outcome="success",} 1.0
graphql_timer_query_seconds_max{operation="execution",operationName="foo",} 0.649467542
graphql_timer_query_seconds_max{operation="parsing",operationName="foo",} 0.022232541
graphql_timer_query_seconds_max{operation="validation",operationName="foo",} 0.014214667
graphql_timer_query_seconds_count{operation="execution",operationName="foo",} 1.0
graphql_timer_query_seconds_sum{operation="execution",operationName="foo",} 0.649467542
graphql_timer_query_seconds_count{operation="parsing",operationName="foo",} 1.0
graphql_timer_query_seconds_sum{operation="parsing",operationName="foo",} 0.022232541
graphql_timer_query_seconds_count{operation="validation",operationName="foo",} 1.0
graphql_timer_query_seconds_sum{operation="validation",operationName="foo",} 0.014214667
```

これらのメトリクスを使うことで、クエリの数・クエリの成功率・クエリの *平均レイテンシ* を計算できます。

# デモ用リポジトリ

https://github.com/itkq/graphql-spring-boot-monitoring

# Prometheus でメトリクスを集める

Prometheus の事例はたくさんあるので詳細は割愛します。Micrometer のメトリクスエンドポイントを叩くと分かりますが、GraphQL 以外にも多くのメトリクスが出力されていることがわかります。不要なものは収集しても仕方がないので、[relabel_config](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#relabel_config) を使って drop するのがいいでしょう。

# Grafana で可視化

例えば次のようなダッシュボードを作れます。

![](/images/graphql-spring-boot-monitoring/grafana1.png)

エラーレートが上がったとき、どのオペレーションか絞り込むことも可能です。

![](/images/graphql-spring-boot-monitoring/grafana2.png)


# 今後の課題

## よりよいメトリクスへ

今回実装したメトリクスにはいくつか課題があります。例えば次のような課題です。

- レイテンシは平均しか計算できない
  - `management.metrics.distribution.percentiles-histogram.graphql = true` にすればパーセンタイルヒストグラムで吐いてくれる。しかしラベル量があまりに多くなり難しい
- 複数クエリをちゃんと考慮できていない
  - 複数クエリを同時に投げてあるクエリが失敗した場合、全体で失敗だと判定する
- カーディナリティを考慮できていない
  - アプリケーションとモニタリングシステムの負担の観点。たとえば graphql-dgs-spring-boot-micrometer では、しきい値を超えたら `others` のように丸める機能がある[^2]

## プロキシで GraphQL を解釈する

クラウドネイティブなプロキシとして有名な Envoy Proxy には Filter という機能があり、リクエストを処理する前後で別の処理を差し込むことができます。例えば [DynamoDB filter](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/dynamodb_filter#config-http-filters-dynamo) は、DynamoDB API のリクエストとレスポンスを解釈し、オペレーションやテーブルレベルの API 統計メトリクスを出力できます。これと同様に GraphQL filter を実装すれば、各 GraphQL サーバにメトリクスを実装することなく、プロキシレイヤでメトリクスを収集できる可能性があります。複数クエリは解釈できるのかなどよく分かっていない点もあります。軽く調べた感じでは事例が見つかりませんでした。


[^1]: https://spring.io/blog/2018/03/16/micrometer-spring-boot-2-s-new-application-metrics-collector
[^2]: https://netflix.github.io/dgs/advanced/instrumentation/#cardinality-limiter
