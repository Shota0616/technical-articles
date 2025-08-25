Linuxコマンド備忘録メモ

# Linux

#### ハードウェア情報を確認

```bash
lshw
```

#### CPU情報を確認

```bash
lscpu
```

#### Linuxのカーネルバージョンとかを確認

```bash
uname -a
```

#### ホスト名の確認

```bash
hostname
```

#### ディストリビューションバージョンの確認


```bash
# ubuntu
cat /etc/os-release

# centos
cat /etc/redhat-release
```

#### ホスト名の変更

```bash
# 一時的にホスト名を変更する
hostname [ホスト名]

# 恒久的にホスト名を変更する
hostnamectl set-hostname [ホスト名]

# 恒久的にホスト名を変更する（ファイルを直接いじる）
vi /etc/hostname
```

#### ディスク確認

```bash
df -h
```
```bash
lsblk
```

#### メモリ確認

```bash
free -h

# swapを使用しているプロセスを確認
grep VmSwap /proc/*/status
```

#### リソースの使用状況を確認

```bash
# システム全体の負荷を確認
top

vmstat 2 # 2s毎

#ディスクI/Oの使用状況
iostat

# psコマンドで確認 (メモリでソート)
ps aux --sort -%mem
```

#### ip確認

```bash
ip a
```

#### エイリアス
```bash
# エイリアス確認
alias

# エイリアス設定
alias hoge="echo hoge"
```

#### 特定サイズのダミーファイルの作成
```bash
# 32MB
dd if=/dev/zero of=test32M bs=1M count=32
```

#### curl
```bash
# curl
curl http://localhost

#各種オプションについて
# メソッドを指定
-X GET
-X POST

# SSLの警告を無視
-k

# リダイレクトする
-L

# 進捗を非表示
-s

# ヘッダー情報を付与
-H

# ダウンロードしたデータをファイル保存
-o xxx.txt

# 実行情報の表示
-w “time_total : %{time_total}”
https://dev.classmethod.jp/articles/curl-benchmark/

# cookieを保存するファイルの指定
-c

# 利用するcookieを指定
-b xxx.txt
```

#### swap領域の作成/削除
```bash
# swap領域の確認
swapon --show
free -h

# swap領域の作成
# 1024MBのswapファイルを作成
dd if=/dev/zero of=/swapfile bs=1M count=1024

# 権限変更
chmod 600 /swapfile

# swapfileをswap領域として設定
mkswap /swapfile

# swapファイルの有効化
swapon /swapfile

# fstabに追加
vi /etc/fstab

/swapfile none swap sw 0 0

# swap領域の無効化
swapoff -a

# 特定のやつを無効化
swapoff /swapfile
```

#### 改行コード変換
```bash
# 改行コードをLFに変換
dos2unix file.txt
```

#### ファイル名検索
```bash
# 全ディレクトリから
find / -name 'failname*'
```

#### 文字コードをutf8に変換
```bash
# 文字コードをUTF-8に変換　-wオプション
nkf -w --overwrite hoge.csv

# 文字コードをShift-JISに変換 -sオプション
nkf -s --overwrite hoge.csv
```

#### DNS

```bash
# オプション特になし
dig [xxxx.hogehoge.com]


# 出力の簡易表示
dig +short [xxxx.hogehoge.com]


# レコードの指定（aレコードの場合）
dig [xxxx.hogehoge.com] a


# dns trace
dig [xxxx.hogehoge.com] +trace
```

#### PATHの確認及び追加
```bash
# 確認
echo $PATH

# PATHを追加するには以下ファイルを編集
vi /root/.bashrc
# 以下の通り記載
export PATH=$PATH:追加したいコマンド検索パス

# sourceで実行
source /root/.bashrc
```

#### linuxユーザのパスワード変更
```bash
# rootユーザになれるとき
passwd [username]

# rootになれなくてカレントのユーザパスワード変えるとき
passwd
```

#### 


# GIT

```bash
# clone
git clone git@github.com:[ユーザ]/[リポジトリ]

# 現在いるブランチの確認
git branch -a

# 現在いるブランチから新しいブランチを切る
git checkout -b [新しいブランチ名]

# ブランチを移動
git checkout [移動先ブランチ名]

# 現在のブランチのコミット履歴を確認（一行で）
git log --oneline

# 現在のブランチのコミット履歴を確認（視覚的に）
git log --graph --oneline --decorate

# リポジトリのリモートのブランチも含んだ最新コミットを表示
git for-each-ref --format='%(authordate) %(refname)' --sort=-committerdate refs/heads refs/remotes

# コミットをリモートにプッシュ
git push origin [リポジトリ名]

# ワークツリーからインデックスへのファイル登録
git add .

# コミットを作成
git commit -m [コミットメッセージ]

# 現在登録されているリモートを確認
git remote -v

# 現在登録されているリモートを削除
git remote rm origin

# リモートを追加
git remote rm origin

# ブランチがどのように移動した確認（mergeしたときのfast-forwardだったかとかの確認ができる）
git reflog

# ブランチ移動
git chekout [移動したいブランチ]

# 現在いるブランチにマージ
git merge origin [マージしたいブランチ]

# fast-forwardのマージだけに限定したい場合
git merge --ff-only origin [ブランチ名]

# マージ取りやめ
git merge –abort

# コミット確認
git show [コミットハッシュ]

## 特定コミット同士の差分確認方法
https://github.com/[ユーザー名]/[リポジトリ名]/compare/[コミットID①]...[コミットID②]

## 特定ブランチ同士の差分確認方法
https://github.com/[ユーザー名]/[リポジトリ名]/compare/[ブランチ名①]...[ブランチ名②]

## 打ち消しのコミット作成
git revert [commit]

## ブランチの削除
git branch -d [ブランチ名]

## 現在の作業ディレクトリとステージングエリアの状態を表示
git status

## diff関連
# ワークツリーの変更点とインデックスとの変更箇所確認
git diff

# git addした後に最新のコミットとの変更点を見たいとき
git diff –cached

# コミット同士を比較する
git diff [変更前のSHA]..[変更後のSHA]

# ブランチ同士
git diff [マージ先ブランチ]...[マージ元ブランチ]

## コミットを打ち消すコミットを作成（--no-editでコミットメッセージを編集しない）
git revert ★コミット★ –no-edit

# 現状のワーキングツリーの修正を消す
git reset –hard

# 直前のコミットを消す（ワーキングツリーの内容はそのまま）
git reset –soft HEAD^

# リセットをリモートに反映（※これは基本的によくない、人のコミット消しかねない）
git push -f

# ファイル権限の確認
git ls-files --stage

# ファイルに権限付与
git update-index --add --chmod=+x [ファイル]

# global config確認
git config --global -l

# global config設定
git config --global core.autocrlf=false

# global config削除
git config --global --unset core.autocrlf

# 変更を一時退避
git stash

# 退避させた変更を復元する
git stash pop

# 一番新しいスタッシュ(stash@{0})の詳細を見る
git stash show -p

# 2番目のスタッシュ(stash@{1})の詳細を見る
git stash show -p stash@{1}
```



# SQL

```sql
-- indexの確認
SHOW INDEX FROM [table_name];

-- index作成
CREATE INDEX index_name ON tbl_name (col_name, ...)

-- index削除
DROP INDEX index_name ON tbl_name;

-- index作成前にcardinality調べたい場合
SELECT COUNT(DISTINCT column_name) AS cardinality FROM table_name;

-- 明示的に使用するindexを指定する
-- FORCE INDEXはindexを使用してテーブル内の行を検索する方法がないという場合以外では指定したindexが使用される。
-- USE INDEXはオプティマイザがindexを使用するよりもテーブルフルスキャンするほうが早いと判断すれば指定したindexは無視される。
FORCE INDEX ([index key])
USE INDEX ([index key])

-- グルーピング (group by)
select * from [table] group by [column]

-- クエリキャッシュのクリア
RESET QUERY CACHE;

-- 外部キーの紐付き確認
show create table [テーブル名];

-- 外部キー制約を一時的に無効化する
SET FOREIGN_KEY_CHECKS = 0;

-- 有効に戻すには
SET FOREIGN_KEY_CHECKS = 1;

-- insert
INSERT INTO テーブル名 (列名1, 列名2,...) VALUES (値1, 値2,...);
-- 列名は省略可能

-- テーブル統計情報の更新
ANALYZE TABLE contract_contract;

-- ストレージエンジンに関する動作情報取得
SHOW ENGINE INNODB STATUS\G;

-- バイナリログのファイル確認
SHOW BINARY LOGS;
SHOW MASTER LOGS;
-- 上記2つのコマンドは等価

-- クエリキャッシュ関連
show status like "%qcache%";
show variables like "%query_cache%";
```


