AWSインスタンスのホスト名が変わってしまっていたときの対処方法

# 概要

AWSのインスタンスを`reboot`したらホスト名が変わってしまうことがあります。
原因としては、↓です。
> Amazon EC2 でインスタンスを起動するとき、起動後にそのインスタンスにユーザーデータを渡し、一般的な自動設定タスクを実行したり、スクリプトを実行したりできます。2 つのタイプのユーザーデータを Amazon EC2 に渡すことができます。シェルスクリプトと cloud-init ディレクティブです。

https://docs.aws.amazon.com/ja_jp/AWSEC2/latest/UserGuide/user-data.html

# 解消方法

### インスタンス起動時に`cloud-config`をユーザーデータを設定する。

1\.インスタンス起動時に高度な詳細を開く
![スクリーンショット 2024-06-08 23.07.01.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/b169f38f-8639-30da-55a6-392ca64ef410.png)
2\.ユーザーデータに以下のように`cloud-config`を追加
![スクリーンショット 2024-06-08 23.10.50.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/5b4d1838-f5f7-a2f3-f3ec-9268298cfc59.png)
```
#cloud-config
fqdn: ★ホスト名★
```

### 起動時にユーザーデータの設定を忘れた場合は、`/etc/cloud/cloud.cfg`の設定を変更する。

1\.変わってしまったホスト名を戻す
```
vi /etc/hostname
```
2\.cloud.cfgの修正 false → true
```
vi /etc/cloud/cloud.cfg
```
```
preserve_hostname: true
```
3\.rebootしてみてホスト名が変わらないか確認
```
reboot
```
