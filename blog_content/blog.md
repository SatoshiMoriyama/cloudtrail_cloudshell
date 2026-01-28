## はじめに

こんにちは。

アプリケーションサービス本部ディベロップメントサービス１課の森山です。

最近、「AWS CloudShell 上で実行された S3 操作をどうやって追跡するか」という課題が出てきたので調査してみました。

AWS CloudShell は便利なブラウザベースのシェル環境です。

CloudShell で投入したコマンド自体は CloudTrail、CloudWatch Logs 等に自動保管されないことは知っていましたが、詳細を把握していませんでした。

本記事では、CloudTrail を使って CloudShell の操作や CloudShell 経由の S3 操作をどのように検知・記録できるのかを調査しました。

### この記事で学べること

- CloudShell 起動時に記録される CloudTrail イベントの種類(一部)
- CloudShell 経由の S3 データイベントを CloudTrail Lake で検知する方法
- userAgent から CloudShell 経由であることを特定する方法
- CloudShell からのファイルダウンロード操作の検知方法

### 前提知識・条件

- CloudShell、CloudTrail、S3 の基本的な知識
- 2026 年 1 月時点の内容となります
- CloudTrail Lake の利用には料金が発生するので、下記をご確認ください
  - https://aws.amazon.com/jp/cloudtrail/pricing/

## 結論

結論は以下です。

- CloudShell の イベントソース`cloudshell.amazonaws.com` として記録される
  - `CreateSession` 等のイベントから誰が使用したかを追跡可能
- CloudShell 上での S3 操作ログは イベントソース`s3.amazonaws.com` として他の操作と同様な形で保持される
  - データイベントについては、CloudTrail Lake や証跡機能の利用が必要
- `s3.amazonaws.com` イベント内の `userAgent` を見れば CloudShell 経由かを判別可能
  - `exec-env/CloudShell` という文字列が含まれている
- コマンド文字列（`aws s3 cp` など）やコマンドの実行結果は CloudTrail に記録されない
  - `.bash_history` はユーザー本人のみアクセス可能
  - CloudShell のホームディレクトリは 120 日間未使用で自動削除される
- コマンドレベルの監査が必要な場合は AWS Systems Manager Session Managerを使用する
  - CloudWatch Logs または S3 にセッションログを記録可能

## やってみた

では、実際に動作確認してみます。

### 準備(AWS CloudTrail Lake の準備)

CloudShell のイベントについては、特に事前準備は不要で、マネジメントコンソール上の `イベント履歴` から過去 90 日分のイベントは確認可能です。

ただし、S3 のデータイベントについては上記画面からは確認できないため、今回は AWS CloudTrail Lake の機能を使って確認します。


[https://docs.aws.amazon.com/ja_jp/awscloudtrail/latest/userguide/cloudtrail-lake.html:embed:cite]

簡単に作成可能なので、詳細な作成手順は割愛します。

保持するイベントを設定する箇所にて、データイベントを有効にし、リソースタイプが `S3` を保持する設定にしておく必要があります。

![alt text](<CleanShot 2026-01-28 at 06.42.23@2x.png>)

![alt text](<CleanShot 2026-01-28 at 06.42.34@2x.png>)

### CloudShell を開く

では、実際に動作確認をしていきます。

まずは、CloudShell を開いてみます。

CloudShell は 2 通りの起動方法があり、それぞれコンソールの表示方式が違うのでお好みの表示方法を選んでください。

#### 1. 画面左端から起動

こちらから起動すると画面下部にコンソールが表示されます。

![alt text](<CleanShot 2026-01-27 at 04.54.36@2x.png>)

#### 2. サービス名から起動

こちらから起動すると画面全体にコンソールが表示されます。

![alt text](<CleanShot 2026-01-27 at 04.55.41@2x.png>)

起動した時点で、コンソール上から `cloudshell.amazonaws.com` のイベントを確認してみたところ、以下のような結果になりました（発生日時の降順で並んでいます）。

![alt text](<CleanShot 2026-01-28 at 05.07.33@2x.png>)

各種イベントの詳細は以下です。

| イベント名 | 説明 |
| --- | --- |
| `PutCredentials` | AWS Management Console へのログインに使用した認証情報が CloudShell に転送されたときに発生 |
| `GetEnvironmentStatus` | CloudShell 環境のステータスが取得されたときに発生（読み取り専用） |
| `CreateSession` | AWS Management Console から CloudShell 環境に接続したときに発生 |
| `RedeemCode` | CloudShell 環境でリフレッシュトークンを取得するワークフローが開始されたときに発生（読み取り専用） |
| `CreateEnvironment` | CloudShell 環境が作成されたときに発生 |
| `DescribeEnvironments` | 環境の詳細を取得 |

参考: https://docs.aws.amazon.com/cloudshell/latest/userguide/logging-and-monitoring.html

今回は CloudShell の環境を全て削除した状態での接続なので、すでに環境がある場合などは出力されるイベントは異なります。

誰かが CloudShell を使ったかという確認をするのであれば、`CreateSession` のイベントレコードを確認すれば良さそうです。

以下、`CreateSession` のイベントレコード内の userIdentity の値を載せておきます。

```json
{
    "eventVersion": "1.09",
    "userIdentity": {
        "type": "AssumedRole",
        "principalId": "AROAXXXXXXXXXX:user@example.com",
        "arn": "arn:aws:sts::123456789012:assumed-role/AWSReservedSSO_AdministratorAccess_xxxxxxxxxxxx/user@example.com",
        "accountId": "123456789012",
        "accessKeyId": "ASIAXXXXXXXXXX",
        "sessionContext": {
            "sessionIssuer": {
                "type": "Role",
                "principalId": "AROAXXXXXXXXXX",
                "arn": "arn:aws:iam::123456789012:role/aws-reserved/sso.amazonaws.com/ap-northeast-1/AWSReservedSSO_AdministratorAccess_xxxxxxxxxxxx",
                "accountId": "123456789012",
                "userName": "AWSReservedSSO_AdministratorAccess_xxxxxxxxxxxx"
            },
            "attributes": {
                "creationDate": "2026-01-27T20:03:35Z",
                "mfaAuthenticated": "false"
            }
        }
    }
}
```

またCloudShell上で任意のコマンドを投入しても、それを記録するようなイベントは出力されていません。

コマンド履歴を確認する場合は、`.bash_history` を確認する、または AWS Systems Manager Session Managerを利用する必要があります。

https://docs.aws.amazon.com/ja_jp/systems-manager/latest/userguide/session-manager.html

### CloudShell 上で S3 コマンドを実行してみる

次に以下のコマンドを CloudShell 上から投入し、任意のファイルをダウンロードしてみます。

```bash
aws s3 cp s3://[任意のファイル]
```

投入後、CloudTrail Lake で以下のクエリを実行し、S3 のデータイベントを確認してみます。

```sql
SELECT 
    userIdentity.principalId,
    eventTime,
    eventName,
    userAgent,
    errorCode,
    errorMessage,
    element_at(requestParameters, 'bucketName') AS bucket,
    element_at(requestParameters, 'key') AS key
FROM <your-event-data-store-id>
WHERE 
    eventCategory = 'Data'
    AND eventSource IN ('s3.amazonaws.com', 's3-control.amazonaws.com')
    AND eventTime >= '2026-01-27 20:00:00'
    AND eventTime <= '2026-01-27 20:59:59'
ORDER BY eventTime DESC
```

※ `<your-event-data-store-id>` は、作成したイベントデータストアの ID

`GetObject`、`HeadObject` のイベントが確認できました。

![alt text](<CleanShot 2026-01-28 at 06.07.47@2x.png>)

それぞれの `userAgent` を確認してみると、以下の値になっています。

```text
aws-cli/2.33.5 md/awscrt#0.29.1 ua/2.1 os/linux#6.1.159-181.297.amzn2023.x86_64 md/arch#x86_64 lang/python#3.13.11 md/pyimpl#CPython exec-env/CloudShell m/Z,b,z,G,E cfg/retry-mode#standard md/installer#exe md/distrib#amzn.2023 md/prompt#off md/command#s3.cp
```

`exec-env/CloudShell` という文字列から CloudShell からの実行であることがわかります。

CloudShell 経由の操作でフィルタリングしたい場合は、`exec-env/CloudShell` の有無をチェックすれば良さそうですね。

### CloudShell からのデータダウンロード

CloudShell には、下記の通り、データをローカルにダウンロード機能があります。

![alt text](<CleanShot 2026-01-28 at 06.32.08@2x.png>)

この操作時には、`GetFileDownloadUrls` イベントが発生しますので、S3 から取得したデータを持ち出していないことの証跡とすることができます。

![alt text](<CleanShot 2026-01-28 at 06.35.33@2x.png>)

## まとめ

以下のようにクエリすれば、CloudShell 経由で S3 のデータを操作した情報を確認できました。

```sql
SELECT 
    userIdentity.principalId,
    eventTime,
    eventName,
    userAgent,
    errorCode,
    errorMessage,
    element_at(requestParameters, 'bucketName') AS bucket,
    element_at(requestParameters, 'key') AS key
FROM <your-event-data-store-id>
WHERE 
    eventCategory = 'Data'
    AND eventSource IN ('s3.amazonaws.com', 's3-control.amazonaws.com')
    AND userAgent LIKE ('%exec-env/CloudShell%')
    AND eventTime >= '2026-01-27 20:00:00'
    AND eventTime <= '2026-01-27 20:59:59'
ORDER BY eventTime DESC
```

さらにコマンド履歴や、`GetFileDownloadUrls` イベントが発生していないことを確認すれば CloudShell 上のみで安全にデータを操作していることを確認できます。

誰かのお役に立てば幸いです。
