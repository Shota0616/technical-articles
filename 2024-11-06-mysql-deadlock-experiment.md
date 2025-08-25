MySQLでわざとデッドロック発生させて挙動を確認してみた

# 概要

RDB（リレーショナルデータベース）を運用していると、複数のトランザクションが同じデータに同時アクセスしようとする場合に「デッドロック」が発生することがあります。デッドロックとは、あるトランザクションが必要とするリソースが別のトランザクションによってロックされ、さらにそのトランザクションも他のリソースのロック解除を待っているため、互いに進行できなくなってしまう状態を指します。

:::note warn
デッドロックが発生すると、データベース全体のパフォーマンスに悪影響を及ぼし、 **最悪の場合は一部の操作が失敗する原因** になります。
:::

文章だけではイメージしにくい場合もあるため、以下の図でデッドロックの概念を視覚的に説明します。


![スクリーンショット 2024-11-04 0.33.44.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/c47d0a62-5809-5c07-6eed-d9723dc953cd.png)


今回はそのデッドロックをわざと発生させて挙動を確認してみます。


## 1. 使用した環境情報

- MySQL: 8.4.3
- エンジン: InnoDB
- Dockerイメージ: mysql:8.4.3

 
https://hub.docker.com/layers/library/mysql/8.4.3/images/sha256-779f2c470d1ee5c84ef7f5fb6514ed4c2ebf659697c8625bc2729be748abc69f?context=explore



## 1. MySQL準備

今回の環境はDokcerコンテナで準備します。

1\. `dokcer-compose.yml`の準備
```yaml:docker-compose.yml
version: '3.8'
services:
  mysql:
    image: mysql:8.4.3
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_USER: user
      MYSQL_PASSWORD: password
    ports:
      - "3306:3306"
```


2\. コンテナ起動
```bash
docker compose up -d
```


3\. プロセス確認及びコンテナに入る
```bash
docker ps
```
```
docker exec -it [container ID] bash
```

4\. MySQLバージョンチェック

```sql
bash-5.1# mysql --version
mysql  Ver 8.4.3 for Linux on x86_64 (MySQL Community Server - GPL)
bash-5.1# 
```



## 2. データの準備


1\. データベース作成
```sql
CREATE DATABASE `testdb`;
USE testdb;
```
2\. テーブル作成
```sql
-- テーブルA作成
CREATE TABLE table_A (
    id INT AUTO_INCREMENT PRIMARY KEY,
    field DECIMAL(10, 2) NOT NULL
);

-- テーブルB作成
CREATE TABLE table_B (
    id INT AUTO_INCREMENT PRIMARY KEY,
    field DECIMAL(10, 2) NOT NULL
);
```
3\. データのインサート
```sql
INSERT INTO table_A (field) VALUES (100.00), (200.00), (300.00);
INSERT INTO table_B (field) VALUES (100.00), (200.00), (300.00);
```
4\. データ確認
```sql
mysql> SELECT * FROM c;
+----+--------+
| id | field  |
+----+--------+
|  1 | 100.00 |
|  2 | 200.00 |
|  3 | 300.00 |
+----+--------+
3 rows in set (0.00 sec)

mysql> SELECT * FROM table_B;
+----+--------+
| id | field  |
+----+--------+
|  1 | 100.00 |
|  2 | 200.00 |
|  3 | 300.00 |
+----+--------+
3 rows in set (0.01 sec)
```

## 3. デッドロックを発生させる

デッドロックは主に 排他ロック と 共有ロック の取り合いから発生します。以下の表は、それぞれのロックの互換性を簡単に示したものです。

|  | 共有ロック | 排他ロック |
|----|----|----|
| 共有ロック | ◯ | ✗ |
| 排他ロック | ✗ | ✗ |

---

### トランザクション1

最初に トランザクション1 の 処理A で table_A の id = 1 に排他ロックをかけてみます。


![スクリーンショット 2024-11-04 12.31.19.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/7d512333-0590-b671-44c7-6b3e20d5f126.png)


1\. トランザクションの開始
```sql
START TRANSACTION;
```

2\. テーブルAのid=1に排他ロックをかけてみる
```sql
use testdb;
SELECT * FROM table_A WHERE id = 1 FOR UPDATE;
```

3\. ロックが取得できているかを確認
```sql
SHOW ENGINE INNODB STATUS;
```
```sql
------------
TRANSACTIONS
------------
Trx id counter 1855
Purge done for trx's n:o < 1854 undo n:o < 0 state: running but idle
History list length 0
LIST OF TRANSACTIONS FOR EACH SESSION:
---TRANSACTION 421755629867008, not started
0 lock struct(s), heap size 1128, 0 row lock(s)
---TRANSACTION 421755629865392, not started
0 lock struct(s), heap size 1128, 0 row lock(s)
---TRANSACTION 421755629864584, not started
0 lock struct(s), heap size 1128, 0 row lock(s)
---TRANSACTION 1854, ACTIVE 6 sec
2 lock struct(s), heap size 1128, 1 row lock(s)
MySQL thread id 13, OS thread handle 140280697189952, query id 108 localhost root User sleep
SELECT *,SLEEP(3600) FROM table_A WHERE id = 1 FOR UPDATE
```

---

### トランザクション2

続いて、 トランザクション2 の 処理B で table_B の id = 1 に排他ロックをかけてみます。


![スクリーンショット 2024-11-04 12.36.56.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/51d5ecfd-473d-4256-3f38-915662d8f14b.png)

1\. トランザクションの開始
```sql
START TRANSACTION;
```

2\. テーブルBのid=1に排他ロックをかけてみる
```sql
use testdb;
SELECT * FROM table_B WHERE id = 1 FOR UPDATE;
```

3\. ロックが取得できているかを確認
```sql
SHOW ENGINE INNODB STATUS;
```
先ほどの結果と比べてトランザクションが増えているのがわかると思います。
```sql
------------
TRANSACTIONS
------------
Trx id counter 1856
Purge done for trx's n:o < 1854 undo n:o < 0 state: running but idle
History list length 0
LIST OF TRANSACTIONS FOR EACH SESSION:
---TRANSACTION 421755629867008, not started
0 lock struct(s), heap size 1128, 0 row lock(s)
---TRANSACTION 421755629865392, not started
0 lock struct(s), heap size 1128, 0 row lock(s)
---TRANSACTION 421755629864584, not started
0 lock struct(s), heap size 1128, 0 row lock(s)
---TRANSACTION 1855, ACTIVE 6 sec
2 lock struct(s), heap size 1128, 1 row lock(s)
MySQL thread id 16, OS thread handle 140280678315584, query id 121 localhost root User sleep
SELECT *,SLEEP(3600) FROM table_B WHERE id = 1 FOR UPDATE
---TRANSACTION 1854, ACTIVE 933 sec
2 lock struct(s), heap size 1128, 1 row lock(s)
MySQL thread id 13, OS thread handle 140280697189952, query id 108 localhost root User sleep
SELECT *,SLEEP(3600) FROM table_A WHERE id = 1 FOR UPDATE
```

---

### デッドロック発生


次に、各トランザクションがさらに処理を進めようとするとデッドロックが発生します。

トランザクション1： 処理B で table_B にアクセスしようとするが、 トランザクション2 によってロックされています。

トランザクション2： 処理A で table_A にアクセスしようとするが、 トランザクション1 によってロックされています。



![スクリーンショット 2024-11-04 0.33.44.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2620245/36515edd-bc2b-555c-f088-ee160cf5e923.png)

実際に処理してデッドロックを発生させてみます。

1\. トランザクション1で処理Bを実行

```sql
SELECT * FROM table_B WHERE id = 1 FOR UPDATE;
```

2\. トランザクション2で処理Aを実行

```sql
SELECT * FROM table_A WHERE id = 1 FOR UPDATE;
```

この操作により、デッドロックが発生し、トランザクション1の処理は以下のように正常に完了します。


```sql
mysql> SELECT * FROM table_B WHERE id = 1 FOR UPDATE;

+----+--------+
| id | field  |
+----+--------+
|  1 | 100.00 |
+----+--------+
1 row in set (10.90 sec)
```

一方で、トランザクション2の処理はエラーとなります。


```sql
mysql> SELECT * FROM table_A WHERE id = 1 FOR UPDATE;
ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
```

# まとめ

デッドロックは、データベースを運用する上で避けられない問題の一つですが、今回のようにわざとデッドロックを発生させることで、実際の挙動やエラーメッセージを確認し、なぜ起こるのかが把握できただけでも怖いものではなくなると思います。

デッドロックを防ぐための一般的な対策として、以下のようなポイントがあります。

* トランザクションの順序を統一する
* トランザクションの粒度を小さくす
* 適切なロックレベルの選択

デッドロックが発生してもデータベースが自動的に検出し、該当するトランザクションを再実行することで影響を最小限に抑えられる場合もありますが、上記のように適切な設計を心がけることが重要です。

