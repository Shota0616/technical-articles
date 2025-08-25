Rundeckの構築方法

# 概要

https://docs.rundeck.com/docs/

> Rundeckとは、PagerDuty, Inc. によって開発されたジョブ管理のソフトウェアである。 ジョブの作成、実行、管理、スケジューリング等をすることができる。 Rundeckは、Javaで動作するため多くのOSで利用可能である。
https://www.designet.co.jp/faq/term/?id=UnVuZGVjaw

一言でいうとcronの進化系かなと思います。

# 構築手順

https://docs.rundeck.com/docs/administration/install/linux-deb.html#installing-rundeck

1\.javaインストール
```
sudo apt-get install openjdk-11-jre-headless
```
2\.リポジトリ署名キーをインポート
```
curl -L https://packages.rundeck.com/pagerduty/rundeck/gpgkey | sudo apt-key add -
```
3\.rundeckインストール
```
sudo apt-get update
sudo apt-get install rundeck
```
4\.daemon-reload及びrundeck起動
```
sudo systemctl daemon-reload
sudo service rundeckd start
```
5\.ログ確認
```
tail -f /var/log/rundeck/service.log
```
6\.以下のようなログが出力されればOK
```
Grails application running at http://localhost:4440 in environment: production
```
7\.今回localhostではないところで動かしているので、以下設定を変更する
```
vi /etc/rundeck/rundeck-config.properties 
```
```
#grails.serverURL=http://localhost:4440
grails.serverURL=http://192.168.11.14:4440
```
1\.rundeck再起動
```
sudo service rundeckd restart
```

http://192.168.11.14:4440/

初期アカウント
ユーザー名 : admin
パスワード : admin



![スクリーンショット 2024-06-11 22.16.02.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/16a64655-c724-5ebe-c3e8-2daae1d13e74.png)

