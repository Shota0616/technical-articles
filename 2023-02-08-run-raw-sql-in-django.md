mysqldumpを使用したバックアップ・リストア

## 概要

mysqldumpを使用したバックアップ・リストア方法についてコマンドを備忘録として残しておきます。今回は、バックアップファイルの容量も考慮して圧縮する方法で実施します。

## バックアップ

普通に`mysqldump`でバックアップを取得して、パイプでつなぎ圧縮も一緒にしていまいます。サーバのストレージに優しいです。

```
# database指定でバックアップ ＆ 圧縮
mysqldump -uusername -ppassword db_name | gzip > /tmp/db-back.gz

# table指定
mysqldump -uusername -ppassword db_name table_name | gzip > /tmp/db-back_table.gz
```

## リストア

先程取得したdumpをリストアしていきます。リストア用のコマンドというのは存在しないので、sqlファイルをmysqlに読み込ませる。

```
# database
zcat /tmp/db-back.gz | mysql -uusername -ppassword db_name

# table
zcat /tmp/db-back_table.gz | mysql -uusername -ppassword db_name
```
※`mysql>`は、mysqlに入って実行する。
