---
title: "incident.ioによるインシデントレスポンス"
emoji: "🌟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["incidentio"]
published: false
publication_name: "ubie_dev"
---

:::message
この記事は[Ubie Engineers & Designers Advent Calendar 2022](https://adventar.org/calendars/7781) 8日目の記事です
:::

Ubie DiscoveryでSREをしているitkqです。Ubieでは、[incident.io](https://incident.io/)を2021年10月頃から導入し、Slackを中心としたインシデントレスポンスを行っています。incident.ioの特徴と、Ubieでの導入から具体的な活用方法を説明します。

# incident.ioとは

https://incident.io/

イギリスのスタートアップが作っている、インシデントレスポンスを行うSaaSです。

## 特徴
- Slackに大きく依存している
  - インシデント単位でWar roomとなるSlackチャンネルが立つ（e.g. #inc-YYYY-MM-DD-title）
  - 基本的にSlackコマンドから操作（サマリのアップデートやアクションの追加など）を行う。`/inc` がエントリーポイントとなっており、コマンドを覚えていなくても操作が容易い
  - War roomチャンネルに書き込んだ情報が、連携により自動で記録される（GitHub issues/pull requests, Sentry etc）
  - ステータスアップデートやフォローアップのリマインド機能
- インシデント単位だけでなく、横断的なものも含め直感的で使いやすいWeb UIがある
- カスタマイズが豊富。テンプレートやロールを変更できる
- SeverityによるGroup byなど、組み込みのインシデントの分析機能がある
- API経由でインシデント作成できたり、インシデント情報を取得できる。組み込みの連携機能もある（PagerDuty, Atlassian Statuspage etc）
- 自動化のための柔軟なワークフローが使える。インシデント発生時やロールのアサイン時などのイベントをトリガーとして、用意されたアクションを設定できる
- テスト用インシデントが機能として組み込まれている（＝通常のインシデントと分けて扱える）
- Community Slackがちゃんと機能している。バグ報告から修正はもちろん、Feature requestの議論もしたことがある

## スクリーンショット

![Slack commands](/images/incident-response-with-incidentio/slack-commands.png =500x)
*/incコマンド。サブコマンドもあるが覚えずとも利用できる*
![UI example](/images/incident-response-with-incidentio/inc-example.png =800x)
*インシデントのWeb UI*
![GitHub Attachment](/images/incident-response-with-incidentio/attach-github.png =500x)
*GitHub連携*


# 導入のきっかけ

元々、インシデントレスポンスのために[Squadcast](https://www.squadcast.com/)というサービスを使っていました。しかしWar roomのSlack連携が無くなったり、組み込みのステータスページは細かいところに手が届かないなど機能面の課題がありました。フィードバックも行っていましたが、プロダクト成長の方向性が自分たちの求めるものと異なっていたことから、移行先を探していました。そこでincident.ioを見つけ、トライアルしたところ非常に使い心地が良く、本導入することにしました。

# 導入前に考えたこと

incident.ioは英語圏のスタートアップが運営するサービスです。当然SlackコマンドのdescriptionやWeb UIは英語で、日本語ローカライズは期待できません。インシデントレスポンスでは、エンジニアだけでなくユーザーサポートなど様々な人が関わるため、日本語非対応なことは障壁になり得ます。しかし、日本で開発している相応のサービスを見つけられず、一方グローバル市場で戦うサービスのほうが機能面の成長に期待できると考えていました。元から利用していたSquadcastも英語のサービスだったこともあり、incident.ioについても英語は慣れでなんとかするという意思決定をしました。

他にも、Slackが中心となるため、インシデント発生と同時にSlackが利用できなくなると致命的です。Slackが使えない場合のコミュニケーションツールとして、別途Google Chatを利用可能にしてはいます。ただしそのような非常事態はまだ発生しておらず、現時点で十分対策できているとは言えません（今後の課題）。


# Ubieにおける導入から活用まで

Ubieでincident.ioの導入から活用までにしたことを振り返ってみます。

## 1. 検証期

### テストインシデントを何回かやる

「Mark as Test incident」ボタンを押すだけでテスト扱いとなり本物のインシデントと区別できるので、ドキュメント読みつつ機能を色々試しました。

![First incident](/images/incident-response-with-incidentio/inc-test.png =600x)
*INC-1*


### SRE付近の人々にも共有

基盤系のセキュリティやデータの人々にもこれ良さそう〜と言ってみたりデモをしたりしました。反応は良かった気がします。

### 軽微なプロダクト障害で活用
当時、影響範囲が少ないバグなどが発生した場合、多様なSlackチャンネルのスレッドで議論が起こり、後からSREがメンションされていました。SRE視点では議論のキャッチアップが難しかったり、スレッド進行では視認性が悪く、また適切に人々を巻き込むのが難しいという課題感がありました。

そこで、そのようなタイミングでincident.ioに起票するよう促し、適宜人々を巻き込むようにしました。インシデントの話をする場所を集約しただけで、SREの認知負荷や、透明性向上を実感できました。また、ユーザーサポートチームとの連携もスムーズになりました。

## 2. 導入期
### 導入に向けた下準備
SREのプラクティスと、incident.ioの機能を見比べながら、インシデントリードなどの役割を整理しました。他にもCustom fieldsにProductを足したり、インシデント作成時に自動的に招待するSlackグループの設定をしました。

### ドキュメントの作成と社内周知
最初はSREがインシデントリードを引き受ける想定はしつつ、SRE以外の人も起票できることを目指し、ドキュメントを作成しました。「どういうときに起票するのか」「起票してまずやること」など最低限のことをまとめました。全社があつまる場で、Squadcastの廃止とincident.ioへの移行を共有しました。

![Incident response document](/images/incident-response-with-incidentio/doc.png =700x)
*インシデントレスポンスのドキュメント (一部抜粋)*
    
### フォローアップの定期確認とポストモーテムの民主化
incident.ioではインシデント横断でフォローアップ進捗を確認できます。この機能を使い、SREの定例で進捗をチェックしつつ、適宜リマインドするようにしました。

また、以前からSRE中心にポストモーテムを行ってはいましたが、incident.ioでのプロセスにも組み込み、正式なものとしました。Notionのデータベースとテンプレートを使ってドキュメントを書き、incident.io上のインシデントと紐付けるようにしました。合わせて「ポストモーテムの心構え」的なドキュメントも作成し、ポストモーテムを行う前に読み合わせるプロセスも足しました（初めての参加者がいる場合）。徐々にファシリテーターをSRE以外のメンバーにも担当してもらうようにしました。

![Postmortem document](/images/incident-response-with-incidentio/postmortem-onboarding.png =700x)
*ポストモーテムのドキュメント (一部抜粋)*
![Postmortem example](/images/incident-response-with-incidentio/postmortem-example.png =600x)
*ポストモーテムの例 (一部抜粋)*


## 3. 活用期

### スコープの拡大
Custom fieldsに項目を増やしつつ、セキュリティインシデントや端末紛失といったコーポレートインシデントもincident.ioで一元的に扱うことにしました。Privateインシデントの機能があり、特にコーポレートインシデントでは活用するようにしています。

### メトリクス分析
ちょうどFour keys基盤の構築をしていたので、Time to Restore ServicesとChange Failure Rateを計算にincident.ioを使うことにしました。[APIからインシデント情報をBigQueryに貯める簡単なコード](https://github.com/itkq/incidentio-to-bq)を書き、GitHubの情報を合わせてdbtで集計し、Redash上に可視化しています。

### エスカレーション体制
各事業におけるインシデントリード、プロダクトや機能単位であるFirst responder、共通基盤やインフラなどのSecond responderを定義し、プロダクトやSeverityに応じてエスカレーションする体制を整えました。また、Opsgenieと連携し、システム的な電話呼び出しに対応しました。

### オンボーディング強化や避難訓練の実施
今後も人員が増加していくことを考えると、スケーラブルなオンボーディングがあるとより嬉しいです。そこでインシデント訓練動画を撮影し、KnowBe4から配信して、視聴してもらうようにしました。

また、あるインシデントが発生した想定で、通話しながらincident.ioを操作する避難訓練の実施も始めています。

# まとめ

インシデントレスポンスをincident.ioで再構築したことで、主に以下が得られました。

- Slackを中心としたプロセスの統一化
- 役割の明確化や、ワークフローで提携作業を自動化、調査や修正をアクションとして明瞭化したことで、インシデント解決までの時間を短縮
- インシデントに対するフォローアップの進捗が明確に
- インシデントに対する透明性の担保

[ペパボさんの事例](https://event.cloudnativedays.jp/cndt2021/talks/1260)のように、ユースケースにあったシステムを自作することも選択肢としては考えられます。しかし、特にプロダクトの提供価値に集中したいスタートアップでは、incident.ioのようなSaaSの活用が現実的です。incident.ioはサービスの機能成長が非常に早く、また成長の方向性が自分たちの理想に近いため、支払うコスト以上の価値を享受できていると感じます。今後もアラートから自動インシデント起票など、さらなる活用を考えています。
