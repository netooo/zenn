---
title: "AWS Glue で 異なるアカウントの RDS から OpenSearch にデータを同期する"
emoji: "🔀"
type: "tech"
topics: ["aws", "glue", "rds", "opensearch"]
publication_name: fukurou_labo
published: false
---

とある開発で RDS の **特定テーブルの特定データのみ** を **日次** で **異なるアカウント** にホスティングしている OpenSearch に同期したいなぁと思うことがありました。
ちょっと特殊なんですが、今回同期したいデータは後から更新されることはなく、いわゆるログデータに近いものになります。
実際に構築してみたんですが、これが思いのほかハマったのでその内容を共有したいと思います。

# やりたいこと
![ideal](/images/migrate_cross_account_rds_to_opensearch_by_glue/ideal.png)

# 構成
最終的には下記の構成になりました。  
Account A に余計なリソースが存在していないのが個人的に気に入ってます。
![architecture](/images/migrate_cross_account_rds_to_opensearch_by_glue/architecture.png)

データ同期であれば Glue でなくても OpenSearch Ingestion や Database Migration Service(DMS) や Lambda を使う方法もありますが、下記の理由で見送りました。

- OpenSearch Ingestion
  - (2025/12/22 時点で) cross-account の RDS に対応してなさそう
  - https://docs.aws.amazon.com/ja_jp/opensearch-service/latest/developerguide/rds-mysql.html#rds-mysql-pipeline-limitations
- DMS
  - 日次同期で十分であり、ニアリアルタイムでの同期は不要(コストを抑えたい)
  - 差分同期も今回は不要
  - アカウントA に replication instance といった余計なリソースを置きたくない
- Lambda
  - 出来るけど、どうせなら Glue で ETL したいじゃん

# 実践
## 1. VPC Peering
VPC や Subnet は既に存在している前提で進めます。

### 1.1 アカウントB で VPC Peering を作成
まずは VPC 間の通信を可能にするため、VPC Peering を作成します。  
どちらのアカウントをリクエスタにしても良いです。  
今回はアカウントB をリクエスタ、アカウントA をアクセプタにします。

ピアリング先は アカウントA の VPC になるため、「別のアカウント」を選択し、接続したい「アカウントID」と「VPC ID」を入力します。
![create_vpc_peering](/images/migrate_cross_account_rds_to_opensearch_by_glue/create_vpc_peering.png)

### 1.2 アカウントA で VPC Peering を承認
アカウントB で申請した VPC Peering の承認を行います。
![accept_vpc_peering](/images/migrate_cross_account_rds_to_opensearch_by_glue/accept_vpc_peering.png)

VPC の **ピアリング接続** を開くと、上記のようにアカウントB で申請した内容が表示されているので、右上のアクションから「リクエストを承諾」を選択しましょう。  
確認画面が表示されるので、問題なければ「リクエストを承諾」します。

### 1.3 DNS 解決を有効化
VPC Peering を申請->承諾しただけでは、Private IP アドレスでの通信は行えません。  
両アカウントで DNS 解決を有効化する必要があります。  
承諾した直後のアカウントA の画面では下記のようになっているはずなので、右上の「DNS 設定を編集」を押下します。
![enable_dns_resolve](/images/migrate_cross_account_rds_to_opensearch_by_glue/enable_dns_resolve.png)

チェック項目にチェックを入れて「変更を保存」しましょう。
![save_dns_resolve](/images/migrate_cross_account_rds_to_opensearch_by_glue/save_dns_resolve.png)

:::message  
リクエスタ側のアカウントB でも同様に DNS 解決を有効化します。
:::

### 1.4 ルートテーブルの設定
これでもまだ VPC 間の通信は行えません。  
上記の有効化はあくまで別アカウントの VPC 内リソースに対して DNS での名前解決ができるようになっただけです。  
ルートテーブルを設定し、名前解決で得た Private IP アドレスに向けた通信を正しくルーティングしましょう。  
この設定も両方のアカウントで設定する必要があります。

まずはアカウントA から設定します。
アカウントB の VPC CIDR は 172.80.0.0/16 であるため、下記のようにルートテーブルを設定します。
![set_route_table_in_account_a](/images/migrate_cross_account_rds_to_opensearch_by_glue/set_route_table_in_account_a.png)

アカウントB でも同様にアカウントA の VPC CIDR をルーティングするように設定します。  
アカウントA の VPC CIDR は 172.31.0.0/16 であるため、下記のようにルートテーブルを設定します。
![set_route_table_in_account_b](/images/migrate_cross_account_rds_to_opensearch_by_glue/set_route_table_in_account_b.png)

これでようやく、VPC Peering による通信が可能になりました。

## 2. OpenSearch
:::message  
今回、使用するエンジンは Managed Cluster の OpenSearch 3.3 です。  
Elasticsearch 7.10 でも正常に動作することを確認しているんですが、 Serverless に関しては未検証です。  
:::

ここからはアカウントB での作業になります。

### 2.1 Security Group を作成
OpenSearch 用の Security Group を作成します。  
インバウンドルールは後に設定するので、今はルール無しで OK です。
![create_sg_for_opensearch](/images/migrate_cross_account_rds_to_opensearch_by_glue/create_sg_for_opensearch.png)

### 2.2 ドメインを作成
OpenSearch ドメインを作成します。  
エンジンバージョンは OpenSearch 3.3 を選択し、 **互換モードを必ずON** にしてください。  
(自分はこれを設定せず、超タイムロスしました。。。)  
ネットワーク設定では任意の Private Subnet に配置し、2.1で作成した Security Group をアタッチします。  
きめ細かなアクセスコントロールでは **マスターユーザー** を設定しました。
![create_opensearch_domain](/images/migrate_cross_account_rds_to_opensearch_by_glue/create_opensearch_domain.png)

## 3. Glue(Data Source)
いよいよ本丸の Glue を作成します。  
まずはアカウントA の RDS をデータソースとするための設定を行います。  
この作業もアカウントB で行いましょう。

### 3.1 Security Group を作成
Glue Connection 用の Security Group を作成します。  
インバウンドルールには自身からの通信を許可するために、すべてのTCPで自身のセキュリティグループを設定します。  
アウトバウンドルールにも自身のセキュリティグループと、OpenSearch にアタッチした Security Group を設定させ、、、  
るのが適切なんですが、4.3で後述する AWS Glue Connector for Elasticsearch の ECR Image Pull が少し特殊なため、今回はすべてのトラフィックを許可する設定にしました。
![create_sg_for_glue_connection_1](/images/migrate_cross_account_rds_to_opensearch_by_glue/create_sg_for_glue_connection_1.png)
![create_sg_for_glue_connection_2](/images/migrate_cross_account_rds_to_opensearch_by_glue/create_sg_for_glue_connection_2.png)

アカウントA の RDS にアタッチしている Security Group に、今回作成した Glue Connection 用の Security Group からのアクセスを許可する Ingress ルールの追加も忘れずに。
![add_ingress_for_glue_connection_sg](/images/migrate_cross_account_rds_to_opensearch_by_glue/add_ingress_for_glue_connection_sg.png)


### 3.2 Glue Connection を作成
アカウントA の RDS に接続するための Glue Connection を作成します。  
データソースを選択する際、「Amazon Aurora」や「MySQL」などを選択せず、「JDBC」を選択します。  
Connection URL は JDBC の形式で入力しましょう。
```
jdbc:mysql://<DB_ENDPOINT>:<DB_PORT>/<DB_NAME>?enabledTLSProtocols=TLSv1.2
```

ネットワーク設定は OpenSearch と同じ Private Subnet に配置し、先ほど作成した Glue Connection 用の Security Group をアタッチします。  
Credential type はお好みで。

![create_glue_connection](/images/migrate_cross_account_rds_to_opensearch_by_glue/create_glue_connection.png)

### 3.3 Glue Crawler を作成
アカウントA の RDS のメタデータを Glue Data Catalog に登録するため、Glue Crawler を作成します。  
ここに関しては同期したいデータの特性によって Crawler 無しでも良いと思います。  
Crawler を使う場合は ETL ジョブのデータソースが Data Catalog となり、使わない場合は Glue Connection となります。  
また Glue Database も事前に作成しておきましょう。
![create_glue_crawler](/images/migrate_cross_account_rds_to_opensearch_by_glue/create_glue_crawler.png)

アタッチする IAM Role の信頼ポリシーは下記のように設定します。
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "glue.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

ポリシーは下記のように設定します。
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "GlueAccess",
      "Effect": "Allow",
      "Action": [
        "glue:Get*",
        "glue:CreateTable"
      ],
      "Resource": [
        "arn:aws:glue:ap-northeast-1:000000000000:catalog",
        "arn:aws:glue:ap-northeast-1:000000000000:connection/*",
        "arn:aws:glue:ap-northeast-1:000000000000:database/*",
        "arn:aws:glue:ap-northeast-1:000000000000:table/*"
      ]
    },
    {
      "Sid": "VpcAccess",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeSubnets",
        "ec2:DescribeVpcEndpoints",
        "ec2:DescribeSecurityGroups",
        "ec2:DescribeRouteTables",
        "ec2:DescribeNetworkInterfaces",
        "ec2:CreateNetworkInterface",
        "ec2:DeleteNetworkInterface",
        "ec2:CreateTags"
      ],
      "Resource": [
        "*"
      ]
    },
    {
      "Sid": "CloudWatchLogsAccess",
      "Effect": "Allow",
      "Action": [
        "logs:PutLogEvents"
      ],
      "Resource": [
        "arn:aws:logs:ap-northeast-1:000000000000:log-group:*"
      ]
    }
  ]
}
```

## 4. Glue(Data Target)
OpenSearch をデータターゲットとするための設定を行います。

### 4.1 Secrets Manager を作成
OpenSearch のマスターユーザーの「ユーザー名/パスワード」を Secrets Manager に登録します。  
ユーザー名のキーは `es.net.http.auth.user` 、パスワードのキーは `es.net.http.auth.pass` としてください。
![create_secrets_manager](/images/migrate_cross_account_rds_to_opensearch_by_glue/create_secrets_manager.png)

### 4.2 AWS Glue Connector for Elasticsearch をサブスクライブ
Glue から OpenSearch にデータを書き込むために [AWS Glue Connector for Elasticsearch](https://aws.amazon.com/marketplace/pp/prodview-v5ygernwn2gb6) を利用します。  
AWS Marketplace で `glue connector for elasticsearch` と検索すると一番上に出てくるので、詳細ページから「購入オプションを表示」を押下し、サブスクライブします。
![subscribe_glue_connector_for_elasticsearch_1](/images/migrate_cross_account_rds_to_opensearch_by_glue/subscribe_glue_connector_for_elasticsearch_1.png)
![subscribe_glue_connector_for_elasticsearch_2](/images/migrate_cross_account_rds_to_opensearch_by_glue/subscribe_glue_connector_for_elasticsearch_2.png)
![subscribe_glue_connector_for_elasticsearch_3](/images/migrate_cross_account_rds_to_opensearch_by_glue/subscribe_glue_connector_for_elasticsearch_3.png)

### 4.3 Glue Connector を作成
AWS Glue Connector for Elasticsearch をサブスクライブすると、AWS Marketplace の「サブスクリプションを管理」に `AWS Glue Connector for Elasticsearch` が追加されます。
![add_glue_connector](/images/migrate_cross_account_rds_to_opensearch_by_glue/add_glue_connector.png)

製品を選択すると「アクティブな契約」にレコードが表示されているので、契約ID を押下し詳細ページを表示します。
![show_glue_connector_details](/images/migrate_cross_account_rds_to_opensearch_by_glue/show_glue_connector_details.png)

右上の「アクション」から「さらにソフトウェアを起動する」を選択します。
![launch_glue_connector](/images/migrate_cross_account_rds_to_opensearch_by_glue/launch_glue_connector.png)

サービスは「ECS」を選択し、配信オプションは「Glue 3.0」を選択します。  
そうすると「起動」の部分に `AWS Glue StudioからGlueコネクタを有効にしてください` とテキストリンクが表示されるので押下します。
![enable_glue_connector](/images/migrate_cross_account_rds_to_opensearch_by_glue/enable_glue_connector.png)

Glue Connector の作成ページにリダイレクトするので、必要な情報を入力し作成します。
- 「Connection access」は 4.1 で作成した Secrets Manager を選択します
- 「Network options」は OpenSearch と同じ Private Subnet に配置し、3.1 で作成した Security Group を選択します

![create_glue_connector](/images/migrate_cross_account_rds_to_opensearch_by_glue/create_glue_connector.png)

### 4.4 NAT Gateway を作成
4.3 で使用した AWS Glue Connector for Elasticsearch は「AWS Account ID: 709825985650」の `us-east-1` リージョンの ECR Image を使用しているため、Private Subnet 内の Glue ジョブから別アカウント・別リージョンの ECR Image を Pull する必要があります。  
そのため ECR 用の VPC Endpoint を作成しても、Private Subnet が ap-northeast-1 リージョンの場合はアクセス出来ません。  
今回は NAT Gateway を Public Subnet に配置して、Image を Pull できるようにしました。  
:::message  
普段 IaC で NAT Gateway を構築していると、久々の手動構築の際、NAT 配置後にルートテーブルの設定を忘れてしまって無駄にハマってしまいました。。。  
ルートテーブルの設定変更も忘れずに!!  
:::

### 4.5 Glue ジョブ用 IAM Role を作成
Glue ジョブ用の IAM Role を作成します。  
信頼ポリシーは下記のように設定します。  
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "glue.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

ポリシーは下記のように設定します。  
ちょっと粗い部分もあるので、細かい部分はお好みで。
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "OpenSearchAccess",
      "Effect": "Allow",
      "Action": [
        "es:*"
      ],
      "Resource": [
        "arn:aws:es:ap-northeast-1:000000000000:domain/zenn-opensearch"
      ]
    },
    {
      "Sid": "GlueAccess",
      "Effect": "Allow",
      "Action": [
        "glue:Get*"
      ],
      "Resource": [
        "arn:aws:glue:ap-northeast-1:000000000000:catalog",
        "arn:aws:glue:ap-northeast-1:000000000000:connection/*",
        "arn:aws:glue:ap-northeast-1:000000000000:database/*",
        "arn:aws:glue:ap-northeast-1:000000000000:table/*"
      ]
    },
    {
      "Sid": "EcrAccess",
      "Effect": "Allow",
      "Action": [
        "ecr:*"
      ],
      "Resource": [
        "*"
      ]
    },
    {
      "Sid": "S3Access",
      "Effect": "Allow",
      "Action": [
        "s3:Get*"
      ],
      "Resource": [
        "arn:aws:s3:::*"
      ]
    },
    {
      "Sid": "SecretsManagerAccess",
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": [
        "arn:aws:secretsmanager:ap-northeast-1:000000000000:secret:zenn-opensearch-credentials"
      ]
    },
    {
      "Sid": "VpcAccess",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeSubnets",
        "ec2:DescribeVpcEndpoints",
        "ec2:DescribeSecurityGroups",
        "ec2:DescribeRouteTables",
        "ec2:DescribeNetworkInterfaces",
        "ec2:CreateNetworkInterface",
        "ec2:DeleteNetworkInterface",
        "ec2:CreateTags"
      ],
      "Resource": [
        "*"
      ]
    }
  ]
}
```

### 4.6 VPC Endpoint for S3 を作成
Glue ジョブでは VPC内から S3 へのアクセスが発生するため、Gateway 型の VPC Endpoint を作成します。
![create_vpc_endpoint_for_s3](/images/migrate_cross_account_rds_to_opensearch_by_glue/create_vpc_endpoint_for_s3.png)

### 4.7 Glue ジョブを作成(Data Source)
いよいよ Glue ジョブを作成します。  
Crawler を用いて Glue Data Catalog にメタデータを登録している場合は「Data Catalog」をデータソースとして設定しましょう。  
「Database」と「Table」を選択し、「IAM Role」には 4.5 で作成した IAM Role を選択します。  
なお今回は最低限の設定で構築するため、諸々のパラメータは必要に応じて設定してください。
![create_glue_job_for_data_source](/images/migrate_cross_account_rds_to_opensearch_by_glue/create_glue_job_for_data_source.png)

画面下部の「Start session」を押下すると、アカウントA の RDS のデータが確認できると思います。

### 4.8 Glue ジョブを作成(Transform)
せっかく Glue を使うのであれば ETL を最大限に活用したいですよね。  
ということで適当に Transform を追加します。  
今回は `id`, `created_at`, `updated_at` のみを抽出するようにしています。
![create_glue_job_for_transform](/images/migrate_cross_account_rds_to_opensearch_by_glue/create_glue_job_for_transform.png)

### 4.9 Glue ジョブを作成(Data Target)
最後にデータターゲットを設定します。  
今回のターゲットは OpenSearch になるので、Data Target は「Amazon OpenSearch Service」、、、とはなりません。  
ここでは 4.3 で作成した Connector を使いたいので、「Elasticsearch Connector」を選択します。

![create_glue_job_for_data_target](/images/migrate_cross_account_rds_to_opensearch_by_glue/create_glue_job_for_data_target.png)

OpenSearch へのアクセスでは「Connection options」の設定が重要になってきます。  
最低限、下記の項目は必要です。

- `es.nodes.wan.only`: true
- `es.nodes`: OpenSearch のドメインエンドポイント (VPC)
- `es.port`: 443
- `es.resource`: <INDEX_NAME>

![set_glue_job_connection_options](/images/migrate_cross_account_rds_to_opensearch_by_glue/set_glue_job_connection_options.png)

### 4.10 OpenSearch のセキュリティグループを更新
このまま Glue ジョブを実行しても `network/Elasticsearch cluster is not accessible or ...` というエラーが発生します。  
はい、そうです、2.1 で作成した OpenSearch のセキュリティグループに Glue Connection にアタッチしたセキュリティグループからのアクセスを許可する必要があります。  
ということでインバウンドルールに Glue 側のセキュリティグループを追加。
![update_sg_for_opensearch](/images/migrate_cross_account_rds_to_opensearch_by_glue/update_sg_for_opensearch.png)

## 5. 動作確認
### 5.1 Glue ジョブの実行
時は来ました。 Glue ジョブを手動で実行しましょう。  
Run Status が `Succeeded` になり、OpenSearch にデータが登録されていれば成功です。  
Glue Job Schedules で日次実行のスケジュールを設定すれば完了!!
![check_glue_job_1](/images/migrate_cross_account_rds_to_opensearch_by_glue/check_glue_job_1.png)
![check_glue_job_2](/images/migrate_cross_account_rds_to_opensearch_by_glue/check_glue_job_2.png)

# 最後に
中々遭遇しないケースかもしれませんが、Glue で cross-account な RDS -> OpenSearch のデータ同期方法をご紹介しました。  
Glue Connector for Elasticsearch 周りでハマるポイントが多いので、同じような構成を検討されている方の参考になれば幸いです。  
ちなみに本来は OpenSearch に収集したデータをアレコレ活用する内容を紹介しようと思っていたので、その記事はまた今度。
