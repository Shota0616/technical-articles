一定期間のメモリ使用率を取得する方法（freeコマンド）

## 概要

一定期間内のメモリの使用率を知りたい事があったので、freeコマンドをリアルタイムで実行しつつファイルに出力する方法で、メモリ使用率を得ました。
ALTER TABLE実行時のメモリ使用率を出力するためにシェルスクリプトを用いていい感じに取得しました。

## 使用するシェルスクリプトの作成

ファイルの作成
```
vi /usr/bin/free.sh
```
ファイルの中身
```sh
#!/bin/bash
set -e

echo `date '+%y/%m/%d %H:%M:%S'`[INFO]メモリ使用率計測処理開始

#freeコマンド実行,メガバイトで、3秒に１回表示,タイムスタンプもつけてます
free -m -s 3 | awk '{ print strftime("%Y/%m/%d %H:%M:%S"), $0 } { system(":") }' > /tmp/free-original.log &
pid1=$!

#####################################################################################
###                ここは適宜実行時にメモリ利用率を計算したいコマンドを記載                     ###
#####################################################################################

#今回はALTER TABLEを実行しているときのメモリ使用率を計測したかったので、以下のような感じ
mysql -u root -e "SELECT NOW();" >> /tmp/result.log
mysql mydb -u root -vvv -e "ALTER TABLE t1 ADD test_coulumn CHAR(1) NOT NULL DEFAULT 'N';" >> /tmp/result.log &
pid2=$!
mysql -u root -e "SELECT NOW();" >> /tmp/result.log

#今回テストのため10秒遅延
sleep 10

#####################################################################################
###                ここは適宜実行時にメモリ利用率を計算したいコマンドを記載                     ###
#####################################################################################

echo `date '+%y/%m/%d %H:%M:%S'`[INFO]コマンド実行中...

#コマンドが実行するまでwait
wait ${pid2}

#freeコマンドをkill
kill -9 ${pid1}

#メモリの使用率をfree結果から抜き出す
awk '/Mem/ {total+=$4;available+=$9} END {printf("%.5f" ,(((total-available)/total)*100))}' /tmp/free-original.log > /tmp/free-mem.log

echo `date '+%y/%m/%d %H:%M:%S'`[INFO]メモリ使用率計測処理終了
```
コマンド実行
```
bash /usr/bin/free.sh
```

## 生成されるファイルについて

生成されるファイルの説明はそれぞれ以下になります。
|  ファイル名  |  用途  |
| ---- | ---- |
|  /tmp/free-original.log  |  freeコマンド実行結果のオリジナルの結果が出力されます  |
|  /tmp/result.log  |  コマンドの標準出力の結果が出力されます  |
|  /tmp/free-mem.log  |  コマンド実行時のメモリ使用率が出力されます  |
|  /usr/bin/free.sh  |  これは生成されるというか先程作ったshファイルです  |

## シェルスクリプトの内容を説明

`set -e`でエラーの場合処理を中止してくれる。
```sh
#!/bin/bash
set -e
``` 
freeコマンドを実行する。メガバイト表示で3秒に１回表示するようにしています。タイムスタンプも表示するようにしていて、その結果は`/tmp/free-original.log`に出力されるようになっています。また、`&`をつけてバックグラウンドで実行するようにしています。
```
free -m -s 3 | awk '{ print strftime("%Y/%m/%d %H:%M:%S"), $0 } { system(":") }' > /tmp/free-original.log &
pid1=$!
```
ここは計測したいコマンドを入力してください。今回は、ALTER TABLEを実行しているときのメモリの使用率を調べたかったので、ALTER TABLEを実行しています。
```
#今回はALTER TABLEを実行しているときのメモリ使用率を計測したかったので、以下のような感じ
mysql -u root -e "SELECT NOW();" >> /tmp/result.log
mysql mydb -u root -vvv -e "ALTER TABLE t1 ADD test_coulumn CHAR(1) NOT NULL DEFAULT 'N';" >> /tmp/result.log &
pid2=$!
mysql -u root -e "SELECT NOW();" >> /tmp/result.log

#今回テストのため10秒遅延
sleep 10
```
上記のコマンドが実行完了するまで、次の処理に行かないようにwaitします。
```
wait ${pid2}
```
コマンドが実行完了したら先ほど実行したfreeコマンドをkillします。
```
kill -9 ${pid1}
```
freeコマンドの実行結果（`/tmp/free-original.log`）からメモリの使用率を計算しています。計算としては、以下になっています。
`メモリ使用率 = (((全体のメモリ使用率 - メモリの空き容量) ÷ 全体のメモリ使用率) × 100)`
```
awk '/Mem/ {total+=$4;available+=$9} END {printf("%.5f" ,(((total-available)/total)*100))}' /tmp/free-original.log > /tmp/free-mem.log
```

# まとめ

今回は、コマンド実行時のメモリ使用率を求めるシェルを作成しました。なにか不備等ありましたらご指摘ください。
