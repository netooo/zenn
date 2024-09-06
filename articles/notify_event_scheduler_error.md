---
title: "MySQL event_scheduler のエラーを通知する"
emoji: "🧤"
type: "tech"
topics: ["mysql", "event_scheduler", "aws", "cloudwatch", "terraform"]
publication_name: fukurou_labo
published: true
---

突然ですが MySQL に [event_scheduler](https://dev.mysql.com/doc/refman/8.0/ja/event-scheduler.html) という機能があるのをご存知でしょうか。
この event_scheduler、指定したクエリを定期的に実行することが出来る機能で cron job のような振る舞いをします。

「1日に1回、特定のクエリを実行したい」というようなシンプルなケースでは、EventBridge Scheduler + Lambda のようなリッチな構成にせずとも実現できるのです。
ただし event_scheduler によるクエリが万が一失敗してしまった場合、どうやって検知すれば良いでしょうか。

ということで今回は(AWS環境で) event_scheduler によるクエリのエラーを検知し通知する方法をご紹介します。

# 構成

シンプルですが、構成図は下記のようになります。

![architecture](/images/notify_event_scheduler_error/architecture.png)

今回の構成では RDS で MySQL が動いており、Slack にエラーを通知します。

まず event_scheduler の実行結果は audit ログに出力されるため、CloudWatch Logs Metric Filter でエラーを拾います。
次に拾ったエラーを CloudWatch Alarm で補足し、SNS Topic に通知します。
最後に SNS Topic から Chatbot 経由で Slack に通知を行います。

# event_scheduler を作成

この記事の本題ではないのでサクッと作成します。

```sql
CREATE EVENT daily_select_hoge
ON SCHEDULE EVERY 1 day
STARTS '2024-09-01 00:00:00'
ON COMPLETION PRESERVE
DO
SELECT * FROM hoge LIMIT 1;
```

この場合、毎日0時に `SELECT * FROM hoge LIMIT 1;` が定期実行されます。

# 通知機構を実装

## 手動

### CloudWatch Logs Metric Filter

audit ログに出力された event_scheduler の結果から、エラーのみを拾う Metric Filter を作成します。
(実は今回の記事のキモはここです。ここが全てと言っても過言ではないです)

![create_metric_filter](/images/notify_event_scheduler_error/create_metric_filter.png)

![create_pattern](/images/notify_event_scheduler_error/create_pattern.png)

`フィルターパターン` の部分を解説します。
まずはじめに、audit ログは event_scheduler 以外のログも流れてくるため、そこから **対象の event_scheduler のログ** を絞り込みます。
それを実現しているのが下記の部分です。

```
"hoge LIMIT 1',"
```

また、audit ログではクエリをシングルクォートで囲っているため、末尾に `'` を付与しています。
さて最後のカンマは何でしょうか。
audit ログでは、クエリ実行によるエラーコードがカンマに続いて返却されます。
クエリに成功した場合は下記のようなログが出力されます。

```
'SELECT * FROM hoge LIMIT 1',0
```

一方でクエリに失敗した場合は下記のように 0 以外のエラーコードが出力されます。

```
'SELECT * FROM hoge LIMIT 1',1064
```

パターンをテストしてみるとイメージしやすいかと思います。

![pattern_test](/images/notify_event_scheduler_error/pattern_test.png)

これを利用し、 **event_scheduler のログかつ成功以外のログ = event_scheduler の失敗** という条件を作成します。
Metric Filter のパターンマッチで成功ログを取り除くには `-` を使用します。
先ほど event_scheduler のログに絞り込んだので、そこから成功ログを取り除くと下記のようになります。

```
"hoge LIMIT 1'," - "hoge LIMIT 1',0"
```

`,0` を取り除くことで、event_scheduler の失敗ログを拾うことが出来ます。
これで event_scheduler の失敗ログを拾う Metric Filter が完成しました。

**待ってください、本当にこれで良いですか...?**

`hoge LIMIT 1` で終わるクエリは event_scheduler 以外で実行されないことを保証できますか？
もし他のクエリが同じ形式で実行された場合、誤検知してしまう可能性があります。

event_scheduler でしか実行されないクエリを保証するために SQL を少し改良します。

```sql
CREATE EVENT daily_select_hoge
ON SCHEDULE EVERY 1 day
STARTS '2024-09-01 00:00:00'
ON COMPLETION PRESERVE
DO
SELECT *
/*daily_select_hoge_event_scheduler*/ FROM hoge LIMIT 1;
```

`/*daily_select_hoge_event_scheduler*/` というコメントを追加しました。
この状態でパターンを

```
"/*daily_select_hoge_event_scheduler*/ FROM hoge LIMIT 1'," - "/*daily_select_hoge_event_scheduler*/ FROM hoge LIMIT 1',0"
```

とすることで event_scheduler でしか実行されないことを保証することができます。
え? なぜ末尾にコメントを追加しないのかって?
私も最初は末尾にコメントを追加して検証してたんですが、末尾だと audit のログ出力時に削除されてしまうんですよね...😅
そのため少し不格好ですが、最終行の先頭にコメントを追加しています。
まぁ実際に設定する event_scheduler のクエリは複数行になると思うので、個人的にはあまり気になりません。

ではメトリクス値を 1 として Metric Filter を作成しましょう。

![preview_metric_filter](/images/notify_event_scheduler_error/preview_metric_filter.png)

### CloudWatch Alarm

上記で作成した Metric Filter をベースに CloudWatch Alarm を作成します。

![create_cloudwatch_alarm](/images/notify_event_scheduler_error/create_cloudwatch_alarm.png)

1回でも event_scheduler のエラーが発生する、つまりメトリクスが 1 以上になった場合に通知を行うようにしたいので、しきい値を 1以上に設定。
アラームを実行するデータポイントも 1/1 とします。

続いて **欠落データ処理** の設定を行います。
基本的に event_scheduler は成功する前提で構築するため、Metric Filter は何も検知しないことがほとんどです。
そのため CloudWatch Alarm は常にデータを「欠落状態」として認識します。
ただし event_scheduler が成功している以上、欠落状態は良好な状態であるため「欠落データを適正(しきい値を超えていない)」を選択します。

![create_cloudwatch_alarm_action](/images/notify_event_scheduler_error/create_cloudwatch_alarm_action.png)

最後に **アクションの通知設定** に Chatbot と連携済みの SNS Topic を指定すれば完成です。

## Terraform

お待たせしました。
こういった設定は全て Terraform で管理したいですね。
今回記事を書くにあたって、初めて手動で設定してみたのですが中々手こずりました...

### CloudWatch Logs Metric Filter

```tf:cloudwatch_logs_metric_filter
locals {
  pattern = "/*daily_select_hoge_event_scheduler*/ FROM hoge LIMIT 1"
}

resource "aws_cloudwatch_log_metric_filter" "this" {
  name           = "DailySelectHogeError"
  log_group_name = "/aws/rds/cluster/${var.rds_cluster_identifier}/audit"
  pattern        = "\"${local.pattern}',\" - \"${local.pattern}',0\""
  metric_transformation {
    name      = "error_event"
    namespace = "event_scheduler/daily_select_hoge"
    value     = "1"
  }
}
```

本来 `pattern` は 

```
"/*daily_select_hoge_event_scheduler*/ FROM hoge LIMIT 1'," - "/*daily_select_hoge_event_scheduler*/ FROM hoge LIMIT 1',0"
```

となるんですが、Terraform では上記全てを 1つの文字列として設定する必要があるため、内側のダブルクォートをエスケープしています。

### CloudWatch Alarm

```tf:cloudwatch_alarm
resource "aws_cloudwatch_metric_alarm" "this" {
  alarm_name                = "DailySelectHogeError"
  alarm_description         = "Notify when daily_select_hoge for event_scheduler error"
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

CloudWatch Logs Metric Filter には metric_transformation を複数設定することが可能ですが、今回は 1 つだけ設定したのでインデックス 0 を指定しています。

:::details Chatbot の Slack 設定
余談ですが、Terraform AWS Provider v5.61.0 より AWS Chatbot の「Slack 設定」と「Teams 設定」がサポートされましたね。
今まで CloudFormation を併用したり、AWS Cloud Control Provider (awscc) を使用するなど煩雑な管理が必要でしたがようやく開放されました🎉

https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/chatbot_slack_channel_configuration
:::

# 通知内容

今回の構成では下記のような内容が Slack に通知されます🙌

![notify_slack](/images/notify_event_scheduler_error/notify_slack.png)

:::message
もし通知内容をカスタマイズしたい場合は、CloudWatch Alarm から直接 SNS Topic に通知するのではなく、EventBridge を経由し [input transformer](https://docs.aws.amazon.com/ja_jp/eventbridge/latest/userguide/eb-transform-target-input.html) を用いると良いと思います。
:::


# 最後に

いかがだったでしょうか。
個人的には満足しつつ、event_scheduler の登録は手動となるので Terraform で管理したいなぁと思ってます💪
マニアックな内容ですが、誰かの参考になれば幸いです!!
