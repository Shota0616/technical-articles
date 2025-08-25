Percona XtraBackupを使用したバックアップの方法について

# 概要
MySQLのバックアップには、一般的には、mysqldumpが使用されていると思います。
mysqldumpは論理バックアップですが、XtraBackupは物理バックアップを採用しているので、高速にバックアップ・リストアが行なえます。
今回は、XtraBackupを導入する手順を紹介します。

:::note warn
XtraBackupの注意点として、バックアップファイルのサイズがmysqldumpと比べて大きくなってしまうというデメリットがあります。
:::

[precona-xtrabackup公式ドキュメント](https://docs.percona.com/percona-xtrabackup/8.0/index.html)

# 環境情報

* `Ubuntu 22.04.3`
* `MySQL 8.0.34`
* `XtraBackup 8.0.34-29`



# XtraBackupのインストール方法

まずは、XtraBackupのインストール手順です。今回ubuntu使用しているので、APTを使用した手順です。

1\. リポジトリのインストール
```
$ wget https://repo.percona.com/apt/percona-release_latest.$(lsb_release -sc)_all.deb
```
2\. ダウンロードしたパッケージをdpkgでインストール
```
$ dpkg -i percona-release_latest.$(lsb_release -sc)_all.deb
```
3\. リポジトリを有効化
```
$ percona-release enable-only tools release
```
4\. パッケージ情報の更新
```
$ apt update
```
5\. XtraBackupのインストール
```
$ apt install percona-xtrabackup-80
```
6\. インストール後のバージョン確認
```
$ xtrabackup --version
xtrabackup version 8.0.34-29 based on MySQL server 8.0.34 Linux (x86_64) (revision id: 5ba706ee)
```
７\.　その他必要なものをインストール
```
$ apt install zstd qpress
```

# バックアップ・リストア方法

以下主に使用する`XtraBackup`コマンドのオプションです。

| オプション | 内容 |
|:-----------|:------------|
| --backup | ターゲットディレクトリにバックアップを作成するモード |
| --prepare | バックアップからデータを復元するモード |
| --user | MySQLユーザ名 |
| --password | MySQLパスワード |
| --socket | ソケットファイルパス |
| --datadir | データディレクトリのパス |
| --compress | 出力データを圧縮するようにする |
| --compress-threads | 圧縮に使用するワーカースレッドの数 |
| --target-dir | バックアップファイルの出力先 |
| --stream | バックアップファイルを指定された形式で標準出力にストリーミングする |

### バックアップ

1\. 今回は、圧縮バックアップを取得してみます。圧縮には、複数スレッドを指定できます。
```
$ mkdir -p /data/compressed
$ xtrabackup --backup --stream=xbstream --compress --compress-threads=4 --target-dir=/data/compressed/ > /data/compressed/backup.xbstream
```
※別途ユーザー名、パスワード、ソケットファイルなどは、適宜指定してください。



2\. 以下のようなログが出ていればバックアップは成功です！
```
2023-10-25T15:03:23.202581-00:00 0 [Note] [MY-011825] [Xtrabackup] completed OK!
```

3\. バックアップファイル出力先を確認してみます。
```
$ ls -lah /data/compressed/
total 1.6M
drwxr-xr-x 2 root root 4.0K Oct 25 15:04 .
drwxr-xr-x 3 root root 4.0K Oct 25 15:01 ..
-rw-r--r-- 1 root root 1.6M Oct 25 15:03 backup.xbstream
```

### リストア

1\. 事前準備（MySQLの停止）
```
$ systemctl status mysql
$ systemctl stop mysql
$ systemctl status mysql
```

2\. 既存のデータディレクトリの移動と新規作成
```
$ mv /var/lib/mysql /tmp/
$ mkdir -p /var/lib/mysql
```

3\. 以下コマンドで、リストア実行
```
$ xbstream -x < /data/compressed/backup.xbstream -C /var/lib/mysql -v
$ xtrabackup --decompress --target-dir=/var/lib/mysql --parallel=4
$ xtrabackup --prepare --target-dir=/var/lib/mysql
```
completedでOK！
```
2023-10-28T10:56:50.201398-00:00 0 [Note] [MY-011825] [Xtrabackup] completed OK!
```

4\. dataディレクトリのファイル権限変更
```
$ chown -R mysql: /var/lib/mysql
$ ls -lhart /var/lib/mysql
```

5\. ステータス変更と変なログ出てないか確認
```
$ systemctl status mysql
$ systemctl restart mysql
$ systemctl status mysql
$ tail /var/log/mysql/error.log
```
問題なさそうです
```
$ systemctl status mysql
● mysql.service - MySQL Community Server
     Loaded: loaded (/lib/systemd/system/mysql.service; enabled; vendor preset: enabled)
     Active: active (running) since Sat 2023-10-28 11:20:26 UTC; 1min 48s ago
    Process: 16346 ExecStartPre=/usr/share/mysql/mysql-systemd-start pre (code=exited, status=0/SUCCESS)
   Main PID: 16354 (mysqld)
     Status: "Server is operational"
      Tasks: 37 (limit: 2221)
     Memory: 356.9M
        CPU: 41.759s
     CGroup: /system.slice/mysql.service
             └─16354 /usr/sbin/mysqld

Oct 28 11:19:44 isoda systemd[1]: Starting MySQL Community Server...
Oct 28 11:20:26 isoda systemd[1]: Started MySQL Community Server.
```

# さいごに

今回、XtraBackupを使用した具体的なバックアップ及びリストア方法について解説しました。パフォーマンスについては触れませんでしたが、後日記事にしようと思います。

