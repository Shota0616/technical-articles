AWS Application Migration Service (AWS MGN) ハンズオンやってみた

# 概要

MGNとは、様々なインフラ基盤で動いているアプリケーションをAWSに<font color="red">**リフトアンドシフト**</font>での移行をサポートしているサービスです。
今回は、`AWS Application Migration Service (MGN)`の[ハンズオン](https://skillbuilder.aws/learn/5BKT7UWR9C/aws-application-migration-service-aws-mgn--a-technical-introduction-/U9MDC2CF51
)で実際にリホストの体験をしてみたいと思います。

:::note info
本記事では、MGNとは何かという話はしません。実際に手を動かしてみてイメージをつかもうという趣旨の記事になっています。
:::


> AWS Application Migration Service を使用すると、物理インフラストラクチャ、VMware vSphere、Microsoft Hyper-V、Amazon Elastic Compute Cloud (Amazon EC2)、Amazon Virtual Private Cloud (Amazon VPC)、およびその他のクラウドから AWS にアプリケーションを移行できます。

MGNのアーキテクチャ概要は以下の通りです。


![スクリーンショット 2025-08-27 8.54.28.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/b73c35e5-cfe1-4b08-8e60-90d363a0e1fe.png)
出典 : [アプリケーション移行サービスのネットワーク要件](https://docs.aws.amazon.com/mgn/latest/ug/preparing-environments.html)


MGNで使用する用語については、以下になります。

| 用語 | 意味 |
|:-|:-|
| レプリケーション設定テンプレート  |  ソースの移行元環境とAWS間でデータをレプリケートするために使用されるレプリケーションサーバ（`Replication Servers`）とステージングエリアのサブネット`Staging Area Subnet`の設定を定義するテンプレート |
|AWS Replication Agent| 移行元のサーバにインストールするエージェントのこと。このエージェントがAWS上のレプリケーションサーバに1500番ポートで、データを送信する。 |
|ソースサーバー| 移行元のサーバのこと。 |
|テストインスタンス| システムの動作の検証を行う、その名の通りテスト用のインスタンス。 |
|カットオーバーインスタンス| 本番運用に使用するインスタンス |
|変換サーバーインスタンス(Application Migration Service Conversion Server)| EBSスナップショットをもとに、ターゲットサーバがAWS上で動作するように変換を行うサーバでMGNによって起動される。主にブートローダーの変更、ハイパーバイザードライバーの挿入、クラウドツールのインストールなどを行う。|


## ステップ0 リージョンの変更

今回のハンズオンでは、`us-east-1`（バージニア北部）のリージョンを使用します。

1\. AWSコンソールにログインして上部右側のリージョンを選択する箇所を押下します。
![スクリーンショット 2025-08-29 11.20.41.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/2ab79ddd-91bb-4df7-b677-c0b1a2e4d535.png)
2\. 選択肢の中から`バージニア北部 us-east-1`を選択します。
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/401c2c47-26bc-4171-ba91-3a6c6116670a.png" width="50%">



## ステップ1 IAMユーザーの作成

まずは、IAMユーザーの作成になります。このIAMユーザーは、「AWS Replication Agent」が使用するためのものです。このエージェントは、移行元のサーバにインストールするものになります。


1\. AWSコンソールから、`IAM > ユーザー > ユーザーの作成`を押下します。
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/ff2cf4a3-8764-4e24-b63d-3dd99b86a47b.png" width="70%">

2\. ユーザー名は「**MGNuser**」とします。AWSマネージメントコンソールへのアクセスは不要なので、チェックは付けなくてOKです。
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/b0d6d1a9-8b18-41f8-a096-1f1e7d11352b.png" width="70%">


3\. 続いて、ポリシーの設定で、「ポリシーを直接アタッチする」を選択し、許可ポリシー「**AWSApplicationMigrationAgentPolicy**」をアタッチします。
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/5f43d491-5c47-48fe-97e7-cfd68e40b39d.png" width="70%">
    <details><summary>AWSApplicationMigrationAgentPolicy ポリシー内容</summary>
    ```json
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "mgn:SendAgentMetricsForMgn",
                    "mgn:SendAgentLogsForMgn",
                    "mgn:SendClientMetricsForMgn",
                    "mgn:SendClientLogsForMgn"
                ],
                "Resource": "*"
            },
            {
                "Effect": "Allow",
                "Action": [
                    "mgn:RegisterAgentForMgn",
                    "mgn:UpdateAgentSourcePropertiesForMgn",
                    "mgn:UpdateAgentReplicationInfoForMgn",
                    "mgn:UpdateAgentConversionInfoForMgn",
                    "mgn:GetAgentInstallationAssetsForMgn",
                    "mgn:GetAgentCommandForMgn",
                    "mgn:GetAgentConfirmedResumeInfoForMgn",
                    "mgn:GetAgentRuntimeConfigurationForMgn",
                    "mgn:UpdateAgentBacklogForMgn",
                    "mgn:GetAgentReplicationInfoForMgn"
                ],
                "Resource": "*"
            },
            {
                "Effect": "Allow",
                "Action": "mgn:TagResource",
                "Resource": "arn:aws:mgn:*:*:source-server/*"
            }
        ]
    }
    ```
    </details>

4\. タグは必要に応じて追加できますが今回は特に必要ないので、そのまま`ユーザーを作成`を押下してユーザーを作成します。

5\. アクセスキーの作成。後続の作業でアクセスキーが必要になるので、作成しておきます。
`IAM > ユーザー > MGNuser`を選択して、`セキュリティ認証情報`のタブに移動します。`アクセスキーを作成`を押下して作成します。別途`アクセスキー`と`シークレットアクセスキー`は控えておいてください。
![スクリーンショット 2025-08-29 12.23.40.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/b9b1d7d7-e8da-41a2-a708-6ae3b5fd03ee.png)



## ステップ2 MGNでの操作

1\. 「**AWS Application Migration Service**」に移動して`使用を開始 > サービスをセットアップ`を押下
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/8e486f4b-e3a1-48eb-bf9d-d05c0eecc633.png" width="70%">

2\. すると自動で`レプリケーション設定テンプレート`が作成されます。`Replication Servers`に使用されるインスタンスタイプやボリュームタイプ、ステージング領域に使用されるサブネットなどをデフォルトから変更したい場合「`Application Migration Service > レプリケーションテンプレート`」を選択し「編集」ボタンを押下することで修正できます。
（今回は、デフォルトの`t3.small`、`SSD (gp3)`とします。）
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/e4dabef1-343c-413e-9fba-794c11ab4887.png" width="70%">


## ステップ3 ソースサーバの作成

:::note info
今回は、ハンズオン用に移行元のサーバを作成しています。
:::

アーキテクチャ図の以下の部分に当たります。
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/a18e324e-8f86-4f60-8372-ad1b8b323919.png" width=40%>

1\. 「`EC2 > AMI`」に移動し、パブリックイメージから「`AWS MGN Workshop Training.`」を検索し、選択します。このイメージにはハンズオン用のWordPressが入っています。
![スクリーンショット 2025-08-29 11.54.14.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/d35be520-372a-411b-9221-4536c7ccfef1.png)

2\. 続いて、「`AMIからインスタンスを起動`」を押下してソース用のサーバを作成していきます。
今回は、以下の通り設定します。（変更箇所だけ記載）
| 項目 | 変更内容 |
|:-|:-|
|  名前(Nameタグ) |  `mgn-workshop-src`という名前をつける |
| キーペア | 既存の鍵か新しく作成する（本来セッションマネージャを使用するべき） |
| セキュリティグループ  | WordPressの画面を確認するため、インバウンドの`80番(http)`を許可。AWS Replication Agentをインストールする必要があるため、インバウンドの`22番(ssh)`を許可|

ちなみに以下の通りブラウザでアクセスするとWordPressの画面が表示されるのがわかると思います。
http://[作成したソースサーバのパブリックIP]
![スクリーンショット 2025-08-29 17.01.43.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/a79ac89e-9fc1-4f65-924a-ed84a232d97f.png)





## ステップ4 ソースサーバに`Replication Agent`のインストール

レプリケーションサーバにソースサーバのデータを送信するため、ソースサーバには`Replication Agent`のインストールが必要になります。

1\. `AWS Replication Agent`のインストールのため、作成したソースのインスタンスにSSHします。
2\. エージェントのインストーラをwgetでダウンロードしていきます。
```
wget -O ./aws-replication-installer-init https://aws-application-migration-service-us-east-1.s3.us-east-1.amazonaws.com/latest/linux/aws-replication-installer-init
```
3\. インストーラを実行します。実行すると以下の通り各種情報を入力する必要があります。（ステップ1で作成したIAMユーザのアクセスキーとシークレットキーを使用します。）
※MGNコンソールの`サーバーを追加`からインストールコマンドを生成することも可能です。
```
sudo chmod +x aws-replication-installer-init; sudo ./aws-replication-installer-init
```
```
[ec2-user@ip-172-31-41-87 ~]$ sudo chmod +x aws-replication-installer-init; sudo ./aws-replication-installer-init
The installation of the AWS Replication Agent has started.
AWS Region Name: us-east-1
AWS Access Key ID: XXXXXXXXXXXXXXXX
AWS Secret Access Key: ****************************************
Identifying volumes for replication.
Choose the disks you want to replicate. Your disks are: /dev/xvda,/dev/nvme0n1
To replicate some of the disks, type the path of the disks, separated with a comma (for example, /dev/sda,/dev/sdb). To replicate all disks, press Enter:
Identified volume for replication: /dev/nvme0n1 of size 12 GiB
All volumes for replication were successfully identified.
Downloading the AWS Replication Agent onto the source server... 
Finished.
Installing the AWS Replication Agent onto the source server... 
Finished.
Syncing the source server with the Application Migration Service Console... 
Finished.
The following is the source server ID: s-3af3a4dfe4d95c45f.
You now have 1 active source server out of a total quota of 150.
Learn more about increasing source servers limit at https://docs.aws.amazon.com/mgn/latest/ug/MGN-service-limits.html
The AWS Replication Agent was successfully installed.
```
> * --region – インストーラーがソースサーバーを登録する AWS リージョン。
> * --aws-access-key-id – インストールユーザーの認証に使用するAWS IAMアクセスキー。このパラメータが指定されていない場合は、インストーラーが入力を求めます。
> * --aws-secret-access-key – インストールユーザーの認証に使用されるAWS IAMアクセスキーに関連付けられたAWS IAMシークレットアクセスキー。このパラメータが指定されていない場合、インストーラーは入力を求めます。
> * --aws-session-token – AWS STS を使用して生成された一時認証情報を使用する際に、セッショントークンが生成されます。一時認証情報を使用する場合、このパラメータを指定しないと、インストーラによって入力を求められます。

4\. すると、MGNのコンソールのソースサーバに追加されていることがわかります。すでに初期同期が自動的に開始されスナップショットの作成中であることも確認できます。
![スクリーンショット 2025-08-29 12.53.27.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/4e378209-cee4-4671-9102-597fc722bca8.png)

5\. `EC2 > インスタンス`を確認すると`AWS Application Migration Service Replication Server`という名前のインスタンスが実行されていることがわかります。
![スクリーンショット 2025-08-29 13.06.16.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/3e9ad78b-6498-4d7e-b46d-1bce11cb07a4.png)
これは、MGNが自動的に立ち上げたインスタンスでソースサーバーにインストールした`Replication Agent`がこの`Replication Server`に対してデータを継続的に送信TCP1500番を使用して送信しています。アーキテクチャ図の以下箇所に当たります。
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/2383a916-c123-4657-b9a3-2d1c68658763.png" width="70%">

6\. しばらくすると以下の通り「テストの準備完了」という状態になります。この状態は、ソースサーバにレプリケーションエージェントがインストールされ、ステージングエリア（レプリケーションサーバ）に最新のデータが反映されている状態です。
![スクリーンショット 2025-08-29 15.56.36.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/870ea816-de5d-45cc-9074-b01abd33c1bb.png)

サーバー情報のタブを見てみると、ソースサーバの情報が色々と表示されています。推奨のインスタンスタイプも表示してくれていて、`c5.large`となっています。（移行元は、`t3.micro`）
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/c3d24980-f47c-4256-8f6e-86ea328465a1.png" width="60%">
`c5.large`と`t3.micro`を比べてみましたが、ちょっとオーバースペックかな・・？
| インスタンス名 | オンデマンドの時間単価 | vCPU | メモリ  | ストレージ | ネットワークパフォーマンス   |
|----------------|------------------------|------|--------|------------|------------------------------|
| t3.micro       | USD 0.0104             | 2    | 1 GiB  | EBS のみ   | 最大 5 ギガビット            |
| c5.large       | USD 0.085              | 2    | 4 GiB  | EBS のみ   | 最大 10 ギガビット           |




## ステップ5 起動テンプレートの編集

テストインスタンス/カットオーバーインスタンスの起動時には、起動テンプレートが使用されます。その起動設定を行っていきます。

1\. MGNのコンソールからソースサーバーを選択して「起動設定」のタブを開きます。
![スクリーンショット 2025-08-29 16.59.16.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/675b775b-b885-4f72-9c36-b3139e951f6c.png)
2\. `一般的な起動設定`の`編集`ボタンを押下します。
![スクリーンショット 2025-08-29 17.03.31.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/e32f2cab-e3ca-498c-9b49-83387afd0181.png)
3\. デフォルトだと`インスタンスタイプの適切なサイズ設定`の部分が`オン`になっているので、`オフ`に変更して「設定を保存」します。（ここがオンの状態だと推奨されたインスタンスタイプがMGNによって勝手に選択されてしまいます。）
![スクリーンショット 2025-08-29 17.10.28.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/90ba2f46-3cb2-43b3-87cc-cca1cd52a282.png)
4\. 続いて、`EC2 起動テンプレート`を`編集`を押下します。そうすると`EC2 > 起動テンプレート > テンプレートを変更 (新しいバージョンを作成)`に遷移します。
![スクリーンショット 2025-08-29 17.15.25.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/cab46100-1fd5-4255-82aa-3879b016090c.png)
5\. `インスタンスタイプ`を`t3.micro`に設定します。
![スクリーンショット 2025-08-29 17.26.23.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/3fe53eff-d035-4343-ac1d-663db3c504f3.png)
6\. `ネットワーク設定 > 高度なネットワーク設定`の`パブリック IP の自動割り当て`を`有効化`に修正し「テンプレートバージョンを作成」を押下し起動テンプレートを作成します。
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/d0666d52-d8b1-41cc-a550-204c22df5f92.png" width="40%">
7\. 先ほど作成した「起動テンプレート」を選択して、「アクション」を押下します。`デフォルトバージョンを設定`を選択し、「テンプレートのバージョン」を`3`に設定します。
![スクリーンショット 2025-08-29 17.31.32.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/e1a7e23e-32bf-4b92-b108-3653cde45304.png)
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/ff702d58-e9a2-4ad5-802c-a800c91b6189.png" width="50%">
8\. 最後にMGNコンソールに戻りインスタンスタイプが`t3.micro`になっていて、テンプレートIDも先ほど作成したものになっているかを確認します。
![スクリーンショット 2025-08-29 17.36.13.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/7191f965-a438-4d2e-95fa-6c7836843e19.png)


## ステップ6 テストインスタンスの起動

続いて、テストインスタンスを起動します。移行ダッシュボードのライフサイクルが「テストの準備完了」という状態になっていればテストインスタンスを起動することができます。
![スクリーンショット 2025-08-30 9.43.50.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/e23926e1-39f6-4e09-9bf0-04f6da423b7a.png)

1\. `テストおよびカットオーバー > テストインスタンスを起動`を押下
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/8fd19e31-e35c-470a-9220-2e22bd0ae908.png" width="40%">
2\. 以下の通り「ジョブの詳細を表示」という部分を押下すると進行中のジョブログが確認できます。また、ソースサーバのダッシュボードに戻ると、移行ライフサイクルが「テスト進行中」となっているかと思います。
![スクリーンショット 2025-08-30 9.47.47.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/6a62e247-7a40-472b-8200-1d33cd7145b3.png)
3\. テストインスタンスを起動を実行すると「AWS Application Migration Service Conversion Server」という名称のインスタンスがMGNによって起動されます。これは、EBSスナップショットをもとに、ターゲットサーバがAWS上で動作するように変換を行うサーバです。
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/3e68446c-113f-421b-b47a-2ff926b43634.png" width="60%">
4\. 移行ダッシュボードに移動して、起動を確認できたら「EC2コンソールで表示」を押下して、テストインスタンスのパブリックIPを確認します。
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/49f8a8a6-7cba-4f47-a664-8cf7e421012b.png" width="50%">
5\. 今回は、http://52.205.213.240 にアクセスして問題なくwordpressの画面が表示されたらテストOKとします。
![スクリーンショット 2025-08-31 15.08.01.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/81629e10-717c-42d3-8dd2-e9762cfcdd9c.png)
6\. テストが完了したら`テストおよびカットオーバー > 「カットオーバーの準備完了」としてマーク`を押下してテストを完了させます。
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/266704d3-3c93-4b8a-ac1f-8a68aef791c3.png" width="40%">
7\. 移行ライフサイクルが「カットオーバーの準備完了」となっていることを確認します。
![スクリーンショット 2025-08-31 15.18.04.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/dccff078-1a8a-4312-b41f-da2a4cd8acae.png)



## ステップ7 カットオーバーインスタンスの起動

ようやく、カットオーバーインスタンスの起動準備が整いました。それでは、やっていきましょう

1\. `テストおよびカットオーバー > カットオーバーインスタンスを起動`を押下し、カットオーバーインスタンスを起動します。
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/d0d7aeed-0c28-4907-94b6-9b00d3f34e1e.png" width="40%">
2\. 移行ダッシュボードに移動して、起動を確認できたら「EC2コンソールで表示」を押下して、テストインスタンスのパブリックIPを確認します。
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/49f8a8a6-7cba-4f47-a664-8cf7e421012b.png" width="50%">
3\. http://54.158.63.225 にアクセスして問題なくwordpressの画面が表示されたらOKです。これで、カットオーバーを完了し移行を完了させることができます。
![スクリーンショット 2025-08-31 15.08.01.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/81629e10-717c-42d3-8dd2-e9762cfcdd9c.png)
4\. `テスト > カットオーバー > カットオーバーを最終処理`を押下し移行プロセスを完了します。
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/e727ff72-e6bf-4665-a53d-103d7f2cd067.png" width="40%">
5\. カットオーバーの最終処理を完了すると、データレプリケーションのステータスが「切断済み」となっていることがわかります。
![スクリーンショット 2025-08-31 15.45.56.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/f8705dbb-90c5-4751-bc23-c8ebd044d5bc.png)
6\. 続いて、`アクション > アーカイブ済みとしてマーク`を押下してソースサーバをアーカイブします。これで一連の移行作業は完了となります。
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/fd7d7f80-cb53-4c06-b98a-1b498f3f07be.png" width="40%">


# まとめ

今回は、様々なインフラ基盤で動いているアプリケーションをAWSに<font color="red">**リフトアンドシフト**</font>での移行をサポートしているサービスのMGNを試してみました。実際に手を動かして移行を実践してみると、かなり理解が深まるかと思います。各種リソースを立てるので多少お金はかかってしまいますが、面白いので是非試してみてください。


