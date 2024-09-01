---
title: "MySQL event_scheduler のエラーを通知する"
emoji: "🧤"
type: "tech"
topics: ["MySQL", "AWS", "CloudWatch", "Terraform", "Tech"]
published: false
---

# MySQLのイベントスケジューラのエラーを拾って通知してみた

## はじめに

突然ですが MySQL にイベントスケジューラという機能があるのをご存知でしょうか？

今回は MySQL のイベントスケジューラで発生したエラーを拾い、通知する仕組みを Terraform で実装してみたので紹介です。

## 事の発端

とある業務で毎日1回特定のクエリを実行したいというケースに遭遇しました。

まぁよくある集計系ですね。

最近インフラ(AWS)にどっぷり浸かっている私は「よし、EventBridge Scheduler と Lambda で作ったろ!」と張り切っていたのですが、

同僚に「MySQL にイベントスケジューラがあるからそれ使えばいいんじゃない？」と言われ面を喰らいました。

[MySQL イベントスケジューラの構成](https://dev.mysql.com/doc/refman/8.0/ja/events-configuration.html)

言われた時はイベントスケジューラの存在すら知らなかったのですが、調べてみると確かに単純な SQL を実行するだけであれば、わざわざ AWS リソースを作成せずともこれでいいじゃん、と。

ただ、イベントスケジューラで設定したクエリが失敗しても通知が来ない点だけはなんとかしないといけなかったので、その辺りを Terraform で実装してみました。

## 構成

シンプルですが、構成図は下記のようになります。

![EventScheduler Architecture](https://github.com/netooo/LT/assets/46105888/2057b99e-0b9e-47a7-984c-d20369746702)

今回のサービスは RDS で MySQL が動いています。

イベントスケジューラの実行結果は audit ログに出力されるため、CloudWatch Logs Metric Filter でエラーを拾います。

拾ったエラーを CloudWatch Alarm で補足し、SNS Topic に通知します。

SNS Topic は Chatbot と連携しており、最終的には Chatbot が Slack 通知を行います。

一例としてイベントスケジューラで実行されるクエリを下記とします。

```sql
SELECT * FROM hoge AS hoge_event_scheduler;
```

```callout
`AS` で別名を付与しているところがポイントです。
```

## Terraform での実装

### CloudWatch Logs Metric Filter

いきなりラスボスなんですが、ここが全てと言っても過言ではないです。

audit ログに出力されたイベントスケジューラの結果から、エラーのみを拾う Metric Filter を作成します。

```terraform
data "aws_cloudwatch_log_group" "this" {
  name = "/aws/rds/cluster/${aws_rds_cluster.this.cluster_identifier}/audit"
}

resource "aws_cloudwatch_log_metric_filter" "this" {
  name           = "EventSchedulerError"
  log_group_name = data.aws_cloudwatch_log_group.this.name
  pattern        = "\"AS hoge_event_scheduler',\" - \"AS hoge_event_scheduler',0\""
  metric_transformation {
    name      = "error_event"
    namespace = "event_scheduler/daily_hoge_select"
    value     = "1"
  }
}
```

`pattern` の部分を解説します。

まずはじめに、audit ログはイベントスケジューラ以外のログも流れてくるため、そこからイベントスケジューラのログを絞り込みます。

それを実現しているのが下記の部分です。

```terraform
pattern = "\"AS hoge_event_scheduler',\""
```

`AS` で別名を付けることで、イベントスケジューラのログを絞り込みやすくしています。

また、audit ログではクエリをシングルクォートで囲っているため、クエリの末尾に `'` を付与しています。

さて最後のカンマは何でしょうか。

audit ログでは、クエリ実行によるステータスコードがカンマに続いて返却されます。

クエリに成功した場合は下記のようなログが出力されます。

```log
'SELECT * FROM hoge AS hoge_event_scheduler',0
```

一方でクエリに失敗した場合は下記のように 0 以外のステータスコードが出力されます。

```log
'SELECT * FROM hoge AS hoge_event_scheduler',1064
```

これを利用し、 **イベントスケジューラのログかつ成功以外のログ=イベントスケジューラの失敗** という条件を作成します。

Metric Filter のパターンマッチで、成功ログを取り除くには `-` を使用します。

先ほどイベントスケジューラのログに絞り込んだので、そこから成功ログを取り除くと下記のようになります。

```terraform
pattern = "\"AS hoge_event_scheduler',\" - \"AS hoge_event_scheduler',0\""
```

これでイベントスケジューラの失敗ログを拾う Metric Filter が完成しました。

あとはこの Metric Filter を CloudWatch Metric に追加します。

```terraform
metric_transformation {
  name      = "error_event"
  namespace = "event_scheduler/daily_hoge_select"
  value     = "1"
}
```

### CloudWatch Alarm

上記で作成した Metric Filter をベースに CloudWatch Alarm を作成します。

```terraform
resource "aws_cloudwatch_metric_alarm" "this" {
  alarm_name                = "DailyHogeSelectError"
  alarm_description         = "Notify when daily_hoge_select error for event_scheduler"
  metric_name               = aws_cloudwatch_log_metric_filter.this.metric_transformation[0].name
  namespace                 = aws_cloudwatch_log_metric_filter.this.metric_transformation[0].namespace
  comparison_operator       = "GreaterThanOrEqualToThreshold"
  threshold                 = "1"
  evaluation_periods        = "1"
  statistic                 = "Sum"
  period                    = "60"
  treat_missing_data        = "notBreaching"
  insufficient_data_actions = []
  alarm_actions             = [var.sns_topic_arn]
}
```

CloudWatch Logs Metric Filter の metric_transformation は複数作成可能ですが、今回は 1 つだけ作成したのでインデックス 0 を指定しています。

1回でもイベントスケジューラのエラーが発生する、つまりメトリクスが 1 以上になった場合に通知を行うようにしたいので下記の設定を行います。

```terraform
comparison_operator = "GreaterThanOrEqualToThreshold"
threshold           = "1"
evaluation_periods  = "1"
statistic           = "Sum"
```

続いて `treat_missing_data` の設定を行います。

基本的にイベントスケジューラは成功する前提で構築するため、Metric Filter は何も検知しないことがほとんどです。

そのため CloudWatch Alarm は常にデータの「欠損状態」として認識します。

ただしイベントスケジューラが成功している以上、データは常に欠損状態であるため、欠損状態を良好状態として扱わせます。

```terraform
treat_missing_data = "notBreaching"
```

最後に `alarm_actions` に Chatbot と連携済みの SNS Topic を指定すれば完成です。

## 最後に

いかがだったでしょうか。

個人的には満足しつつ、欲を言えばイベントスケジューラの登録まで Terraform で完結させることが出来れば 100点だと感じています。

マニアックな内容ですが、誰かの参考になれば幸いです。
