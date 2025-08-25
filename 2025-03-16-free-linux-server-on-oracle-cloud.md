24GB RAM + 4CPU + 200GBストレージのLinuxサーバーを無料で手にいれる方法

# 概要

エンジニアであれば誰もが、無料で利用できるサーバを求めていると思います。
この記事では、`OCI(Oracle Cloud Infrastructure)`を使用して24GB RAM + 4CPU + 200GBストレージのUbuntuサーバを手にいれる方法を詳しく解説していきます。

無料で利用できるパブリッククラウドとしてAWS, GCPも存在しますが、どちらも無料枠は1年間限定で、期限が切れると有料プランに移行してしまいます。しかしOCIについては期限の制限なしに無料枠を提供してくれています。

# Oracle Cloud のAlways Freeクラウド・サービスとは？

期間の制限なくOCIのサービスを使用できるもの。

✅ ARMベースの`4CPU + 24GB RAM`のVMインスタンスを1か月あたり3,000 OCPU時間と18,000GB時間で使用可能。例えば、`4CPU + 24GB RAM`のVM1台を作ることもできるし、`1CPU + 6GB RAM`のVMを4台作ることも可能。
✅ 2つのBlock Volumesストレージ、合計`200GB`

:::note info
3,000 OCPU時間
4OCPUを750時間（約1か月）使うと、3,000OCPU時間

18,000GB時間
24GBのメモリを750時間使うと、18,000GB時間

公式サイトには難しく書かれていますが、実質「4CPU + 24GB RAM + 200GBストレージのサーバーを無料で期間に制限なく使用できる」ということです！
:::

※詳しくは公式サイトを参照お願いします。

https://www.oracle.com/jp/cloud/free/

# ステップ1 Oracle Cloudに登録する

1. OCIアカウントの登録ページへ移動 https://signup.oraclecloud.com/
1. 必要な情報を入力しメールアドレス認証
1. クレジットカード情報の入力 
※クレジットカードの検証のために少額の決済が発生してしまいますが、すぐに返却されるので安心して下さい。

![スクリーンショット 2025-03-12 21.35.27.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/06d06af5-2d87-44a1-84fe-c4197f063b9f.png)

もしもアカウントの作成に詰まったら以下のサイトを参考にしてみると良いかもです。

https://qiita.com/fufukuku/items/af58e01ec0ee93f063b0

# ステップ2 無料のインスタンスを作成

アカウントの作成が完了したらOCIにログインして
1\. 「コンピュート > インスタンス」を選択します。

![スクリーンショット 2025-03-12 22.45.49.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/4fd0fb83-c44d-488e-8de9-9a43d772b0fe.png)


2\. 次に「インスタンスの作成」を選択します。

![スクリーンショット 2025-03-12 22.49.49.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/7cd56806-2d3d-4291-b2ec-cb180be6e935.png)


3\. 次に使用するイメージとシェイプを選択しますが、イメージは無料のものを選択して下さい。シェイプに関しては、以下の赤枠で囲っている「Always Free対象」のものを選択して下さい。

![スクリーンショット 2025-03-12 23.18.00.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/6c5d3c05-1633-4b2e-ac14-eb36d8aa2e7d.png)


4\. 次に「ブートボリューム」を最大200GBで選択します。
5\. 最後に「作成」を押下すればインスタンスの作成が完了します。

# もしインスタンスを作成できない場合

インスタンスを作成しようとしたときに以下のエラーが出て作成できないことがあります。
おそらくOCIが用意しているAlways Freeアカウントで取得可能なリソースを制限しているものだと考えられます。
一部の記事では、自動でトライし続けるスクリプトを組んでいる方もいますが、アカウントのアップグレードすれば作成できる可能性が高いのでそちらも一つの手かもしれません。（コストの管理は注意が必要です。）

:::note warn
可用性ドメインAD-1のシェイプVM.Standard.A1.Flexの容量が不足しています。別の可用性ドメインでインスタンスを作成するか、後で再試行してください。 フォルト・ドメインを指定した場合は、フォルト・ドメインを指定せずにインスタンスを作成します。それでも問題が解決しない場合は、後で再試行してください。
:::

![スクリーンショット 2025-03-14 22.04.05.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/e55ebccf-ec0e-4a3b-b453-35204d1f2847.png)

# まとめ

OCIのAlways Freeを活用すれば24GB RAM + 4CPU + 200GBストレージを無料で使用できることがわかりました。是非この無料枠を活用してみて下さい！

（実際にSSHしてサーバ確認してみた結果）
```
Architecture:             aarch64
  CPU op-mode(s):         32-bit, 64-bit
  Byte Order:             Little Endian
CPU(s):                   4
  On-line CPU(s) list:    0-3
Vendor ID:                ARM
```

```
               total        used        free      shared  buff/cache   available
Mem:            23Gi       298Mi        22Gi       5.0Mi       426Mi        22Gi
Swap:             0B          0B          0B
```

```
/dev/sda1       194G  1.6G  193G   1% /
```
