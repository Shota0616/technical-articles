mysqldumpslowでmysqlのslowクエリを集計する方法（出力結果をcsv化する）

# 概要

mysqlのチューニングでは、slowクエリから潰していくみたいなことをすると思いますが、そのときに便利なmysqldumpslowについて紹介します。

# 事前にデータを用意

slowクエリを出すためのテスト用データを作成します。データは公式のものを使用します。

https://dev.mysql.com/doc/index-other.html

https://github.com/datacharmer/test_db/tree/master

1\. git clone
```
git clone git@github.com:datacharmer/test_db.git
```
2\. sqlファイルを流し込む
```
mysql < employees.sql
```
3\. databaseができたか確認
```
mysql> show databases like "employees";
```
```
+----------------------+
| Database (employees) |
+----------------------+
| employees            |
+----------------------+
1 row in set (0.00 sec)
```

# slowクエリを出力するための設定変更

https://dev.mysql.com/doc/refman/8.0/ja/slow-query-log.html

1\. 以下の通りmysqldの設定を変更する（今回は1s以上のクエリが出力されるように設定）
```
vi /etc/mysql/mysql.conf.d/mysqld.cnf
```
```conf
[mysqld]

slow_query_log=1
slow_query_log_file=/var/log/mysql/mysql-slow.log
long_query_time=1
```
2\. 再起動
```
systemctl restart mysql
```
3\. 出力されるか確認
```
tail /var/log/mysql/mysql-slow.log
```
```
Time                 Id Command    Argument
# Time: 2023-12-16T03:48:55.037837Z
# User@Host: root[root] @ localhost []  Id:     8
# Query_time: 2.556357  Lock_time: 0.000003 Rows_sent: 1  Rows_examined: 2844047
use employees;
SET timestamp=1702698532;
select count(*) from salaries where emp_no like "%12%";
```

# mysqldumpslowを使用したslowクエリの集計

前置きが長くなりましたが、slowクエリの集計方法について説明します。

https://dev.mysql.com/doc/refman/8.0/ja/mysqldumpslow.html

> 通常、mysqldumpslow は数字の特定の値および文字列データ値以外が同様のクエリーをグループ化します。 サマリーの出力を表示する際、これらの値を N および 'S' に「抽象化」します。 値の抽象化動作を変更するには、-a および -n オプションを使用します。

mysqldumpslowでは、クエリが全く同じものがグルーピングされますが、検索値などは抽象化されます。

mysqldumpslowコマンドのオプションは以下の通りです。

|オプション名|説明|
|:-:|:-:|
|-a|文字列と数字の抽象化をしない|
|-n|指定された桁数の数字を抽象化する|
|--debug, -d|デバッグ情報を書き込み|
|-g|(grep形式の)パターンに一致するステートメントのみを表示|
|--help|ヘルプメッセージを表示して終了|
|-h|ログファイル内のサーバホスト名を指定|
|-i|サーバインスタンス名を指定|
|-l|合計時間からロック時間を減算しない|
|-r|ソート順序を逆転|
|-s|出力のソート方法を指定|
|-t|最初から指定された数だけのクエリのみを表示|
|--verbose|上長モード|

`-s c`（カウント数）を指定したときのソートはタイプは以下の様になります。
(例)
```
mysqldumpslow -s c /var/log/mysql/mysql-slow.log
```

|オプション名|説明|
|:-:|:-:|
|t|クエリの総実行時間|
|at|クエリの平均実行時間|
|l|ロックの総時間|
|al|ロックの平均時間|
|r|総送信行数|
|ar|平均送信行数|
|c|カウント（クエリの実行回数）|


実行結果の例を記載しておきます。

```
mysqldumpslow -t 30 -s at /var/log/mysql/mysql-slow.log
```
```
Reading mysql slow query log from /var/log/mysql/mysql-slow.log
Count: 11  Time=2.71s (29s)  Lock=0.00s (0s)  Rows=1.0 (11), root[root]@localhost
  select count(*) from salaries where emp_no like "S" or salary like "S" or from_date like "S"

Count: 9  Time=1.60s (14s)  Lock=0.00s (0s)  Rows=1.0 (9), root[root]@localhost
  select count(*) from salaries where emp_no like "S" or salary like "S"

Count: 1  Time=1.29s (1s)  Lock=0.00s (0s)  Rows=1.0 (1), root[root]@localhost
  select count(*) from salaries where emp_no like "S" or salary

Count: 30  Time=1.19s (35s)  Lock=0.00s (0s)  Rows=1.0 (30), root[root]@localhost
  select count(*) from salaries where emp_no like "S"

Died at /usr/bin/mysqldumpslow line 162, <> chunk 51.
```

# mysqldumpslowの出力結果を整形

mysqldumpslowでslowクエリを集計できましたが、上記のフォーマットのままだと使いにくいので、csv形式に整形します。

https://github.com/Shota0616/mysqlslow-to-csv/blob/main

<details><summary>整形用のshellを表示する</summary><div>

```shell
#!/bin/bash
set -e

###########################################################
# 集計対象ファイル設定
TARGET_FILE_NAME=mysql-slow.log
TARGET_FILE_DIR=/var/log/mysql/
TARGET_FILE_PATH=${TARGET_FILE_DIR}${TARGET_FILE_NAME}
# 集計結果の出力ディレクトリ
OUTPUT_FILE_PATH=/var/log/mysql/mysqldumpslow/
# 本シェルのログ出力設定
OUTPUT_LOG_FILE_NAME=mysqldumpslow-to-csv.log
OUTPUT_LOG_FILE_DIR=/var/log/mysql/
OUTPUT_LOG_FILE_PATH=${OUTPUT_LOG_FILE_DIR}${OUTPUT_LOG_FILE_NAME}
# 集計識別用のuuid生成
UUID=`uuidgen`
###########################################################

# ログ出力ディレクトリの存在チェック
if [ ! -d ${OUTPUT_LOG_FILE_DIR} ]; then
    mkdir -p ${OUTPUT_LOG_FILE_DIR}
fi
if [ ! -e ${TARGET_FILE_PATH} ]; then
    touch ${OUTPUT_LOG_FILE_PATH}
fi
# log start
echo `date '+%y/%m/%d %H:%M:%S'` "INFO[${UUID}]" "[start] - mysqldumpslow-to-csv.sh" >> ${OUTPUT_LOG_FILE_PATH}

# 引数チェック処理
if [ $# != 2 ]; then
    echo `date '+%y/%m/%d %H:%M:%S'` "ERROR[${UUID}]" "[argument] - argument errer" >> ${OUTPUT_LOG_FILE_PATH}
    echo "引数の数が合いません。引数は2つだけ指定してください。"
    exit 1
fi

# 各種ディレクトリ存在チェック
# 対象ファイル
if [ -e ${TARGET_FILE_PATH} ]; then
    echo `date '+%y/%m/%d %H:%M:%S'` "INFO[${UUID}]" "[target_file_check_ok] - mysqldumpslow-to-csv.sh" >> ${OUTPUT_LOG_FILE_PATH}
else
    echo `date '+%y/%m/%d %H:%M:%S'` "ERROR[${UUID}]" "[target_file_check_failed] - target_file_check errer" >> ${OUTPUT_LOG_FILE_PATH}
    echo "対象ファイルが存在しません。"
    exit 1
fi
# 集計結果の出力ディレクトリ
if [ ! -d ${OUTPUT_FILE_PATH} ]; then
    echo `date '+%y/%m/%d %H:%M:%S'` "INFO[${UUID}]" "[make_output_file_dir] - mysqldumpslow-to-csv.sh" >> ${OUTPUT_LOG_FILE_PATH}
    mkdir -p ${OUTPUT_FILE_PATH}
fi

###########################################################
# 取得件数
COUNT=${1:-30}
# 集計オプション
OPTION=${2:-at}
###########################################################





echo `date '+%y/%m/%d %H:%M:%S'` "INFO[${UUID}]" "[start] - mysqldumpslow-cmd-start" >> ${OUTPUT_LOG_FILE_PATH}

# 総実行時間
mysqldumpslow -t ${COUNT} -s ${OPTION} ${TARGET_FILE_PATH} | sed '/^$/d' | awk '/^Count:/ {if (query) print query; query=""; printf $0","; next} {query=query" "$0} END {print query}' | awk '/^Count:/ {if (match($0, /Count: ([0-9]+)  Time=([0-9.]+)s \(([0-9]+)s\)  Lock=([0-9.]+)s \(([0-9]+)s\)  Rows=([0-9.]+) \(([0-9]+)\), (\S+@\S+),   (.*)/, item)) print item[1]","item[2]","item[3]","item[4]","item[5]","item[6]","item[7]","item[8]",\""item[9]"\""}' | sed -e "1i count,time-avg,time-total,lock-avg,lock-total,rows-avg,rows-total,host,statement" > mysqldumpslow-${OPTION}-`date '+%y%m%d-%H%M%S'`.csv

echo `date '+%y/%m/%d %H:%M:%S'` "INFO[${UUID}]" "[finish] - mysqldumpslow-cmd-finish" >> ${OUTPUT_LOG_FILE_PATH}

echo `date '+%y/%m/%d %H:%M:%S'` "INFO[${UUID}]" "[finish] - mysqldumpslow-to-csv.sh" >> ${OUTPUT_LOG_FILE_PATH}
```
</div></details>

上記のshellscriptを使用して、結果をcsvで出力できます。

1\. shellscriptをclone
```
git clone git@github.com:Shota0616/mysqlslow-to-csv.git
```
2\. 一応権限変更しておく
```
cd mysqlslow-to-csv
chmod 755 mysqlslow-to-csv.sh
```
3\. 引数指定して実行する
```
bash mysqlslow-to-csv.sh [TOP◯◯] [ソートタイプ]
```
(例)クエリの実行回数でTOP3まで出力するとき
```
bash mysqlslow-to-csv.sh 3 c
```
4\. すると以下のディレクトリにcsvで出力されています
```
ls -lahrt /var/log/mysql/mysqldumpslow/
```
5\. 中身を確認してみる
```
view /var/log/mysql/mysqldumpslow/mysqldumpslow-c-231217-055108.csv
```
```csvs
count,time-avg,time-total,lock-avg,lock-total,rows-avg,rows-total,host,statement
30,1.19,35,0.00,0,1.0,30,root[root]@localhost,"select count(*) from salaries where emp_no like "S""
11,2.71,29,0.00,0,1.0,11,root[root]@localhost,"select count(*) from salaries where emp_no like "S" or salary like "S" or from_date like "S""
9,1.60,14,0.00,0,1.0,9,root[root]@localhost,"select count(*) from salaries where emp_no like "S" or salary like "S""
```

# まとめ

今回slowクエリの集計方法についてやり方をまとめました。デフォルト出力だと正直使用しにくいので、自分はcsvに変換して使用しています。slowクエリを撲滅することでwebアプリのパフォーマンス向上に役立つと思うので、ぜひ使ってみてください。

