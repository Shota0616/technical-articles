Let's Encryptの無料SSL証明書でhttps化しよう

# 概要

今回は、Let's Encryptの無料SSL証明書を使用してHTTPSを有効化する方法について詳しく解説していきます。
Let's Encryptでは、無料で「ワイルドカード証明書」を発行することも可能です。

https://letsencrypt.org/ja/docs/

:::note info
Let's Encryptは認証局(Certificate Authority; CA)の一つです。
:::

:::note warn
この記事では、証明書を配置したいサーバにSSHできることを前提に書いています。
:::

:::note warn
Let's Encryptの証明書の期限は3ヶ月と短いので、自動更新するようにしましょう。
:::

# 環境

* `certbot 2.7.3`
* `Ubuntu 22.04.3 LTS`
* `nginx/1.18.0 (Ubuntu)`

# Certbotのインストール

[Certbot](https://certbot.eff.org/)というACMEクライアントを使用することによって、証明書の発行とインストールをダウンタイムゼロで自動化できます。
使用環境によってCertbotが要件を満たさない場合は、[その他](https://letsencrypt.org/ja/docs/client-options/)の対応しているACMEクライアントを使用してください。


1\. snapのインストール
```
sudo apt update
sudo apt install snapd
sudo snap install hello-world
```
:::note warn
Certbotのインストールは、snapを使用することが推奨されています。
:::
2\. ステータスチェック
```
systemctl status snapd
```
3\. 他のパッケージマネージャでインストールしたCertbotがある場合は削除
```
例)
sudo apt-get remove certbot
sudo dnf remove certbot
sudo yum remove certbot
```
4\. Certbotインストール
```
sudo snap install --classic certbot
```
5\. シンボリックリンク貼る
```
sudo ln -s /snap/bin/certbot /usr/bin/certbot
```
6\. バージョンチェック
```
certbot --version
```


# 証明書の取得

:::note warn
証明書の検証にHTTP-01チャレンジを使用する場合は、80番ポートを開けておく必要があります。もしもポートが自由に開けられない等の事情がある場合は、DNS-01チャレンジ（後述）を使用してください。
:::

* 証明書を取得してnginx構成を自動的に編集する場合
```
sudo certbot --nginx
```

* ★今回はこっち★
証明書のみを取得する場合（今回は取得のみ実施してインストールは手動でしてみました。）
```
sudo certbot certonly --nginx --agree-tos -d xxxxxxxxxx.com -m xxxxxxxxxx@xxxx.xxx
```
dオプション：ドメイン名
mオプション：メールアドレス
```
Saving debug log to /var/log/letsencrypt/letsencrypt.log

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Would you be willing, once your first certificate is successfully issued, to
share your email address with the Electronic Frontier Foundation, a founding
partner of the Let's Encrypt project and the non-profit organization that
develops Certbot? We'd like to send you email about our work encrypting the web,
EFF news, campaigns, and ways to support digital freedom.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o: N
Account registered.
Requesting a certificate for xxxxxxxxxxx.com

Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/xxxxxxxxxx.com/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/xxxxxxxxxx.com/privkey.pem
This certificate expires on 2024-01-27.
These files will be updated when the certificate renews.
Certbot has set up a scheduled task to automatically renew this certificate in the background.

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
If you like Certbot, please consider supporting our work by:
 * Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
 * Donating to EFF:                    https://eff.org/donate-le
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
```
自動更新がtimerに設定されているかも確認します。
```
systemctl list-timers   
```
```
NEXT                        LEFT               LAST                        PASSED      UNIT                           >
Sun 2023-10-29 23:41:00 UTC 7h left            n/a                         n/a         snap.certbot.renew.timer       >
```

# Nginxに実際に導入

1\. 先程発行した証明書と鍵を確認
```
ls -lah /etc/letsencrypt/live/xxxxxxxxxx.com/
```
2\. ssl用の設定ファイルの作成
```
vi /etc/nginx/conf.d/ssl.conf
```
以下内容で作成
```
server {
	listen 443;
	server_name xxxxxxxxxx.com;
	
	ssl on;
    ssl_certificate     /etc/letsencrypt/live/xxxxxxxxxx.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/xxxxxxxxxx.com/privkey.pem;
}
```
3\. nginxのリロード
```
systemctl reload nginx
systemctl status nginx
```
4\. ブラウザから証明書チェック


# ワイルドカード証明書の発行

Let's Encryptでは、無料でワイルドカード証明書を発行することも可能です。
ワイルドカード証明書とはCNにアスタリスク（*）を含むFQDNを指定することで、複数のサブドメインに対して一つの証明書で対応できるようなものです。

:::note warn
ワイルドカード証明書の検証には、DNS-01チャレンジしか使用できません。
:::

1\. ワイルドカード証明書発行コマンド実行
```
sudo certbot certonly --manual --agree-tos -d *.xxxxxxxxxx.com -m xxxxxxxxxx@xxxx.xxx --server https://acme-v02.api.letsencrypt.org/directory --preferred-challenges dns
```
上記コマンドを実行すると以下のような表示になります。下の通り、`_acme-challenge.xxxxxxxxxxx.com.`のTXTレコードをワイルドカードのドメインを管理しているDNSにトークンを入れて登録してください。
レジストラによっては、DNS登録まで結構時間がかかると思うので、登録が完了するまで待ってください。
（route53は一瞬）
```
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Requesting a certificate for *.xxxxxxxxxxx.com

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Please deploy a DNS TXT record under the name:

_acme-challenge.xxxxxxxxxxx.com.

with the following value:

XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX

Before continuing, verify the TXT record has been deployed. Depending on the DNS
provider, this may take some time, from a few seconds to multiple minutes. You can
check if it has finished deploying with aid of online tools, such as the Google
Admin Toolbox: https://toolbox.googleapps.com/apps/dig/#TXT/_acme-challenge.xxxxxxxxxxx.com.
Look for one or more bolded line(s) below the line ';ANSWER'. It should show the
value(s) you've just added.

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Press Enter to Continue
```
そして、Enterを押下すると
```

Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/xxxxxxxxxxx.com-0001/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/xxxxxxxxxxx.com-0001/privkey.pem
This certificate expires on 2024-01-28.
These files will be updated when the certificate renews.

NEXT STEPS:
- This certificate will not be renewed automatically. Autorenewal of --manual certificates requires the use of an authentication hook script (--manual-auth-hook) but one was not provided. To renew this certificate, repeat this same certbot command before the certificate's expiry date.

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
If you like Certbot, please consider supporting our work by:
 * Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
 * Donating to EFF:                    https://eff.org/donate-le
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
```
成功しました。
`/etc/letsencrypt/live/xxxxxxxxxxx.com-0001/fullchain.pem`に証明書ができているはずです。
Nginxへの導入手順は上記に記載の通りです。

# まとめ

今回は、無料で証明書を導入する手順を説明しました。
httpのサイトは警告がでてしまったりと何かと都合が悪いので、ちゃんと暗号化したほうがいいと思います。
