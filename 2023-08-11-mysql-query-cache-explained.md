MySQLのクエリキャッシュについて詳しくまとめてみた。

## 概要

この記事では、MySQLのクエリキャッシュについて基本的な設定の方法からパラメータチューニングの方法まで詳しく解説していきます。

:::note warn
警告
クエリキャッシュはMySQLで非推奨となり、MySQL 8.0 では削除されました。
MariaDBでは、削除されていないみたいです。
:::
※低いバージョンのMySQLを使用することを余儀なくされている環境では、使用を検討してみてください。

## クエリキャッシュとは

[公式ドキュメント](https://dev.mysql.com/doc/refman/5.7/en/query-cache.html)
> クエリ キャッシュには、ステートメントのテキスト SELECTと、クライアントに送信された対応する結果が格納されます。後で同一のステートメントを受信した場合、サーバーはステートメントを解析して再度実行するのではなく、クエリ キャッシュから結果を取得します。クエリ キャッシュはセッション間で共有されるため、あるクライアントによって生成された結果セットを、別のクライアントによって発行された同じクエリに応答して送信できます。

つまり、同じ構文のステートメントはキャッシュできますよって機能です。静的なサイトの場合かなり効果は見込めると思います。逆に動的なサイトの場合は効果は薄いです。

ちなみに以下のクエリは違うものとみなされます。同一と見なされるには、クエリが完全に同じバイトである必要があるので注意してください。
```sql
SELECT * FROM tbl_name
Select * from tbl_name
```

## 環境

- `MariaDB 11.0.2`
- `Ubuntu 22.04.2 LTS`

## 設定方法

まずは、クエリキャッシュを有効にする方法を解説します。クエリキャッシュが有効になっているかは、以下のコマンドで確認することができます。`query_cache_type`が`OFF`だと無効になっています。

```sql
MariaDB [(none)]> SHOW VARIABLES LIKE '%query_cache%';
+------------------------------+---------+
| Variable_name                | Value   |
+------------------------------+---------+
| have_query_cache             | YES     |
| query_cache_limit            | 1048576 |
| query_cache_min_res_unit     | 4096    |
| query_cache_size             | 1048576 |
| query_cache_strip_comments   | OFF     |
| query_cache_type             | OFF     |
| query_cache_wlock_invalidate | OFF     |
+------------------------------+---------+
7 rows in set (0.001 sec)
```

有効にするには、mariadb-serverの設定ファイルに以下の設定を追加してください。設定を適用するには、再起動が必要になります。（`systemctl restart mariadb`）
そうすると`query_cache_type`が`ON`になります。

```/etc/mysql/mariadb.conf.d/50-server.cnf
[mariadbd]
query_cache_type=1
```
オンラインで適用したい場合は、以下のように設定します。
```
SET GLOBAL query_cache_type = ON;
```
これで、ひとまずクエリキャッシュは有効になりました。しかし、デフォルトだとキャッシュできるクエリの容量とリミットは`1MB`となっているので、すぐに消費してしまいます。ここらへんの数値はチューニングする必要があります。

## 各種パラメータ

クエリキャッシュには、いくつか設定する項目と使用状況を表すステータス変数が存在します。ステータス変数を確認することにより、チューニングが可能になります。

---

#### 設定項目

設定値は、`SHOW VARIABLES LIKE '%query_cache%';`で確認可能です。
```sql
MariaDB [(none)]> SHOW VARIABLES LIKE '%query_cache%';
+------------------------------+---------+
| Variable_name                | Value   |
+------------------------------+---------+
| have_query_cache             | YES     |
| query_cache_limit            | 1048576 |
| query_cache_min_res_unit     | 4096    |
| query_cache_size             | 1048576 |
| query_cache_strip_comments   | OFF     |
| query_cache_type             | ON      |
| query_cache_wlock_invalidate | OFF     |
+------------------------------+---------+
7 rows in set (0.001 sec)
```

それぞれの設定値についての詳細は以下になります。

| 設定値 | 内容 | デフォルト値 |
|:-----------|:------------|:------------|
| have_query_cache       | サーバがクエリキャッシュをサポートしているかどうか。サポートしているときは、YES。サポートしていないときは、NO。 |-|
| query_cache_limit     | クエリキャッシュにキャッシュされるサイズの上限(バイト単位)。これ以上大きい結果のクエリはキャッシュされない。     |1048576 (1MB)|
| query_cache_min_res_unit     | クエリキャッシュ結果に割り当てられるブロックの最小サイズ (バイト単位)       |4096 (4KB)|
| query_cache_size     | クエリキャッシュで使用可能なサイズ (バイト単位) 。クエリキャッシュには約40KBが必要なため、これより小さいサイズを設定すると警告が表示される。      |1048576 (1MB)|
| query_cache_strip_comments     | 1に設定すると、サーバーは検索前にクエリからコメントを削除し、クエリ キャッシュにコメントが存在するかどうかを確認する。複数のスペース、改行、タブ、その他の空白文字も削除される。      |OFF (0)|
| query_cache_type     | 0を設定するとクエリキャッシュは無効になる (ただし、 query_cache_sizeバイトのバッファは割り当てられる)。1を設定するとSQL_NO_CACHEが指定されない限り、すべてのSELECTクエリがキャッシュされる。2に設定するとDEMAND、SQL CACHE句を含むクエリのみがキャッシュされる      |OFF (0)|
| query_cache_wlock_invalidate     | デフォルトの0に設定すると、テーブルに書き込みロックがかかっていても、クエリ・キャッシュにある結果が返される。1に設定すると、クライアントはまずロックが解除されるのを待つ必要がある。      |OFF (0)|

---

#### ステータス変数項目

ステータス変数は、`SHOW STATUS LIKE 'Qcache%';`で確認可能です。
```sql
MariaDB [(none)]> SHOW STATUS LIKE 'Qcache%';
+-------------------------+---------+
| Variable_name           | Value   |
+-------------------------+---------+
| Qcache_free_blocks      | 1       |
| Qcache_free_memory      | 1031272 |
| Qcache_hits             | 0       |
| Qcache_inserts          | 0       |
| Qcache_lowmem_prunes    | 0       |
| Qcache_not_cached       | 28      |
| Qcache_queries_in_cache | 0       |
| Qcache_total_blocks     | 1       |
+-------------------------+---------+
8 rows in set (0.000 sec)
```

それぞれのステータス変数についての詳細は以下になります。

| 設定値 | 内容 |
|:-----------|:------------|
|Qcache_free_blocks|空きメモリブロック数|
|Qcache_free_memory|空きクエリキャッシュメモリ量|
|Qcache_hits|クエリキャッシュによって処理されたリクエストの総数（キャッシュヒット数）|
|Qcache_inserts|これまでにクエリキャッシュにキャッシュされたクエリの総数|
|Qcache_lowmem_prunes|割当メモリ不足によって削除されてしまったキャッシュの総数|
|Qcache_not_cached|クエリキャッシュにキャッシュできない、または、SQL_NO_CACHEを使用したクエリの総数|
|Qcache_queries_in_cache|現在クエリキャッシュにキャッシュされているキャッシュの総数|
|Qcache_total_blocks|クエリキャッシュによって使用されているブロック数|

## クエリキャッシュのチューニングについて

ここからは、クエリキャッシュのチューニング方法について何ステップか分けて解説してきます。

---

#### ステップ1

まずは、以下設定値の最適化を行います。これは、クエリキャッシュにキャッシュできる総量のサイズとキャッシュ可能な1つのクエリのサイズ上限を定めます。

* **query_cache_limit**
* **query_cache_size**

これの決め方ですが、環境によって実行されるクエリも違うので、一度設定してみて、だめなら増やすか減らすかしていく以外に道はありません。

:::note warn
リソースが無限なら一気にsizeを増加させてもいいですが、オーバーヘッドも増える可能性があるので、おすすめしません。
:::

結論から言うと、以下になります。詳細は下記例の部分で説明します。

`Qcache_lowmem_prunes` が0以外 → `query_cache_size`を上げる
`query_cache_size`を限界まであげても`Qcache_lowmem_prunes` が0以外 → `query_cache_limit`を下げる

:::note warn
※上記のステータス変数は、ある程度DBが使用されてから確認してください。`Qcache_lowmem_prunes`の初期値は0なので、ある程度動かさないと割当メモリ不足かを判断できません。
:::

(チューニングの例)

1. 以下の設定を行うとします。

```
query_cache_size=2M
query_cache_limit=1M
```
上記が設定できているか確認します。
```sql
MariaDB [test]> show variables like "%query_cache%";
+------------------------------+---------+
| Variable_name                | Value   |
+------------------------------+---------+
| have_query_cache             | YES     |
| query_cache_limit            | 1048576 |
| query_cache_min_res_unit     | 4096    |
| query_cache_size             | 2097152 |
| query_cache_strip_comments   | OFF     |
| query_cache_type             | ON      |
| query_cache_wlock_invalidate | OFF     |
+------------------------------+---------+
7 rows in set (0.001 sec)
```
`query_cache_size`の値が、2Mになっているので、OKそうです。
この状態で、アプリケーションを運用してみます。ある程度運用したら2に移ります。
<br>
2. クエリキャッシュの使用状況を確認
先程、`Qcache_lowmem_prunes`が0以外のときは、`query_cache_size`を上げるといいました。
`Qcache_lowmem_prunes`はメモリの不足によって削除された数なので、クエリキャッシュに割り当てているメモリが不足していることになります。なので、`query_cache_size`を上げます。
`Qcache_lowmem_prunes`が0になればメモリは十分であることがわかるので、`query_cache_size`は十分ということになります。
```sql
MariaDB [test]> show status like "%Qcache%";
+-------------------------+--------+
| Variable_name           | Value  |
+-------------------------+--------+
| Qcache_free_blocks      | 1      |
| Qcache_free_memory      | 242880 |
| Qcache_hits             | 1      |
| Qcache_inserts          | 11     |
| Qcache_lowmem_prunes    | 3      |
| Qcache_not_cached       | 29     |
| Qcache_queries_in_cache | 7      |
| Qcache_total_blocks     | 18     |
+-------------------------+--------+
8 rows in set (0.000 sec)
```
<br>
3. query_cache_sizeを限界まで上げてもQcache_lowmem_prunesが0にならないとき

リソースは有限なので`query_cache_size`を上げられる量も有限です。全体のサイズを上げきってもどうにもならないときは、キャッシュできるクエリ1つの上限値を下げてあげる必要があります。
なので、`query_cache_size`を上げきっても`Qcache_lowmem_prunes`が0にならないときは、`query_cache_limit`を下げてあげます。

**これで、クエリキャッシュのパラメータの設定は概ねOKです。**

---

#### ステップ2

**ステップ1**で、クエリキャッシュの機能を最大限活用できるようになりました。
**ステップ2**では、クエリキャッシュを適用しているアプリケーションの発行するクエリがクエリキャッシュに適しているかを調査していきます。

クエリキャッシュのヒット率があまりにも低い場合はそのアプリケーションはクエリキャッシュに適していないかもしれません。ヒット率は以下のSQLで確認することができます。
```sql
計算式
(Qcache_hits / (Qcache_hitsQcache_hits + Qcache_inserts + Qcache_not_cached)) × 100

SELECT (SELECT VARIABLE_VALUE FROM INFORMATION_SCHEMA.GLOBAL_STATUS WHERE VARIABLE_NAME = 'QCACHE_HITS')/(SELECT SUM(VARIABLE_VALUE) FROM INFORMATION_SCHEMA.GLOBAL_STATUS WHERE VARIABLE_NAME IN ('QCACHE_HITS','QCACHE_INSERTS','QCACHE_NOT_CACHED'))*100 AS CACHE_HIT_RATE;
```
結果は以下のような形で出てきます。ヒット率が47%なので、あまり良いとは言えません。ここのチューニングはDB側では不可能なので、どうすることもできません。
```sql
MariaDB [test]> SELECT (SELECT VARIABLE_VALUE FROM INFORMATION_SCHEMA.GLOBAL_STATUS WHERE VARIABLE_NAME = 'QCACHE_HITS')/(SELECT SUM(VARIABLE_VALUE) FROM INFORMATION_SCHEMA.GLOBAL_STATUS WHERE VARIABLE_NAME IN ('QCACHE_HITS','QCACHE_INSERTS','QCACHE_NOT_CACHED'))*100 AS CACHE_HIT_RATE;
+-------------------+
| CACHE_HIT_RATE    |
+-------------------+
| 47.61904761904761 |
+-------------------+
1 row in set (0.001 sec)
```

:::note warn
ここでヒット率が極端に低い場合は、アプリケーションの発行するクエリがクエリキャッシュに適していないということになるので、**クエリキャッシュの使用は避けたほうが**良いかもしれません。。無駄なオーバーヘッドが増えてしまいます。
:::

詳細は以下になります。

まず、クエリキャッシュのヒット率を上げるには、ヒット回数（`Qcache_hits`）を上げる必要があります。
では、そのヒット回数を上げるには、どうするべきかというと**キャッシュが削除されない**ことが大切になってきます。

上記を踏まえクエリキャッシュのキャッシュが削除されるタイミングについて知る必要があります。キャッシュが削除されるタイミングは、DB再起動とリセットコマンドを実行したときを除くと以下の2つになります。

* `query_cache_size`不足によるクエリの削除 = Qcache_lowmem_prunes
* キャッシュしているクエリのテーブルが更新されたことによる削除

`query_cache_size`不足によるクエリの削除は、**ステップ1**で解消しました。
では、テーブル更新による削除はDB側で制御できるでしょうか？答えは、NOです。何らかのアプリケーションが発行するSQLの制御はアプリケーション側でしかできないので、DBでは、どうすることもできないです。
なので、`query_cache_size`不足によるクエリの削除を解消してもヒット率が上がらない場合は、どうすることもできないです。（発行されるSQLを調整する）

## Tips

その他クエリキャッシュについて詳細に調査する方法についても残しておきます。

### その他クエリキャッシュ関連のコマンド

* **クエリキャッシュのリセット**
```sql
RESET QUERY CACHE;
```

* **クエリキャッシュのフラッシュ**
クエリキャッシュがメモリを解放するときにフラグメンテーション（断片化）が発生してしまうので、定期的にクリアする必要があります。この断片化したブロックは、`Qcache_free_blocks`であらわされます。
```sql
FLUSH QUERY CACHE;
```

* **キャッシュされているクエリの平均サイズを算出**
現在キャッシュされているクエリの平均サイズを算出します。これを知ることで、query_cache_limitの数値の参考似できます。
```sql
計算式
(query_cache_size - Qcaceh_free_memory) / Qcache_queries_in_cache

SELECT (((SELECT VARIABLE_VALUE FROM INFORMATION_SCHEMA.GLOBAL_VARIABLES WHERE VARIABLE_NAME LIKE 'QUERY_CACHE_SIZE') - (SELECT VARIABLE_VALUE FROM INFORMATION_SCHEMA.GLOBAL_STATUS WHERE VARIABLE_NAME LIKE 'QCACHE_FREE_MEMORY')) / (SELECT VARIABLE_VALUE FROM INFORMATION_SCHEMA.GLOBAL_STATUS WHERE VARIABLE_NAME LIKE 'QCACHE_QUERIES_IN_CACHE')) AS QUERY_SIZE_AVG;
```

* **`query_cache_size`不足によるキャッシュの削除率**
ステップ1で`query_cache_size`を十分に設定していたらこの削除率は0になります。

```sql
計算式
(Qcache_lowmem_prunes / Qcache_inserts) × 100

SELECT ((SELECT VARIABLE_VALUE FROM INFORMATION_SCHEMA.GLOBAL_STATUS WHERE VARIABLE_NAME = 'QCACHE_LOWMEM_PRUNES') / (SELECT VARIABLE_VALUE FROM INFORMATION_SCHEMA.GLOBAL_STATUS WHERE VARIABLE_NAME = 'QCACHE_INSERTS')) * 100 AS 'LOWMEM_DELETE_CACHE_RATE';
```

* **データ更新によるキャッシュの削除率**
キャッシュしているクエリのテーブルにUPDATE,DELETE,INSERTなどの更新があった場合にキャッシュは削除されます。キャッシュがインサートされた数に対してデータ更新によって削除されたクエリの削除率です。
```sql
計算式
((Qcache_inserts - Qcache_lowmem_prunes - Qcache_queries_in_cache) / Qcache_inserts) × 100

SELECT (((SELECT VARIABLE_VALUE FROM INFORMATION_SCHEMA.GLOBAL_STATUS WHERE VARIABLE_NAME = 'QCACHE_INSERTS') - (SELECT VARIABLE_VALUE FROM INFORMATION_SCHEMA.GLOBAL_STATUS WHERE VARIABLE_NAME = 'QCACHE_LOWMEM_PRUNES') - (SELECT VARIABLE_VALUE FROM INFORMATION_SCHEMA.GLOBAL_STATUS WHERE VARIABLE_NAME = 'QCACHE_QUERIES_IN_CACHE')) / (SELECT VARIABLE_VALUE FROM INFORMATION_SCHEMA.GLOBAL_STATUS WHERE VARIABLE_NAME = 'QCACHE_INSERTS')) * 100 AS 'DATAUPDATE_DELETE_CACHE_RATE';
```

* **インサートに対するキャッシュ保持率**
キャッシュインサートに対するキャッシュの保持率です。
```sql
計算式
Qcache_queries_in_cache / Qcache_inserts) × 100

SELECT ((SELECT VARIABLE_VALUE FROM INFORMATION_SCHEMA.GLOBAL_STATUS WHERE VARIABLE_NAME = 'QCACHE_QUERIES_IN_CACHE') / (SELECT VARIABLE_VALUE FROM INFORMATION_SCHEMA.GLOBAL_STATUS WHERE VARIABLE_NAME = 'QCACHE_INSERTS')) * 100 AS 'RETENTION_CACHE_RATE';
```

※インサートに対するキャッシュ保持率 + データ更新によるキャッシュの削除率 + `query_cache_size`不足によるキャッシュの削除率 = 100%


### キャッシュされているクエリの詳細調査方法

キャッシュされているクエリの詳細を調査したいときは、[Query Cache Information Plugin](https://mariadb.com/kb/en/query-cache-information-plugin/)というプラグインを入れることで可能になります。
[表示項目について](https://mariadb.com/kb/en/information-schema-query_cache_info-table/)

プラグイン導入方法
設定ファイル書く方法
```/etc/mysql/mariadb.conf.d/50-server.cnf
[mariadb]
plugin_load_add = query_cache_info
```
オンラインでインストール、アンインストールする方法
```sql
# インストール
INSTALL SONAME 'query_cache_info';

# アンインストール
UNINSTALL SONAME 'query_cache_info';
```

このプラグインを導入することによって、`information_schema`に`QUERY_CACHE_INFO`テーブルが追加されます。
テーブルは以下のような感じになっています。
```sql
MariaDB [information_schema]> DESC QUERY_CACHE_INFO;
+--------------------------+--------------+------+-----+---------+-------+
| Field                    | Type         | Null | Key | Default | Extra |
+--------------------------+--------------+------+-----+---------+-------+
| STATEMENT_SCHEMA         | varchar(192) | NO   |     | NULL    |       |
| STATEMENT_TEXT           | longtext     | NO   |     | NULL    |       |
| RESULT_BLOCKS_COUNT      | int(11)      | NO   |     | NULL    |       |
| RESULT_BLOCKS_SIZE       | bigint(11)   | NO   |     | NULL    |       |
| RESULT_BLOCKS_SIZE_USED  | bigint(11)   | NO   |     | NULL    |       |
| LIMIT                    | bigint(11)   | NO   |     | NULL    |       |
| MAX_SORT_LENGTH          | bigint(11)   | NO   |     | NULL    |       |
| GROUP_CONCAT_MAX_LENGTH  | bigint(11)   | NO   |     | NULL    |       |
| CHARACTER_SET_CLIENT     | varchar(32)  | NO   |     | NULL    |       |
| CHARACTER_SET_RESULT     | varchar(32)  | NO   |     | NULL    |       |
| COLLATION                | varchar(64)  | NO   |     | NULL    |       |
| TIMEZONE                 | varchar(50)  | NO   |     | NULL    |       |
| DEFAULT_WEEK_FORMAT      | int(11)      | NO   |     | NULL    |       |
| DIV_PRECISION_INCREMENT  | int(11)      | NO   |     | NULL    |       |
| SQL_MODE                 | varchar(250) | NO   |     | NULL    |       |
| LC_TIME_NAMES            | varchar(100) | NO   |     | NULL    |       |
| CLIENT_LONG_FLAG         | tinyint(11)  | NO   |     | NULL    |       |
| CLIENT_PROTOCOL_41       | tinyint(11)  | NO   |     | NULL    |       |
| CLIENT_EXTENDED_METADATA | tinyint(11)  | NO   |     | NULL    |       |
| PROTOCOL_TYPE            | tinyint(11)  | NO   |     | NULL    |       |
| MORE_RESULTS_EXISTS      | tinyint(11)  | NO   |     | NULL    |       |
| IN_TRANS                 | tinyint(11)  | NO   |     | NULL    |       |
| AUTOCOMMIT               | tinyint(11)  | NO   |     | NULL    |       |
| PACKET_NUMBER            | tinyint(11)  | NO   |     | NULL    |       |
| HITS                     | bigint(11)   | NO   |     | NULL    |       |
+--------------------------+--------------+------+-----+---------+-------+
25 rows in set (0.001 sec)
```
試しにSELECTしてみます。

```sql
MariaDB [information_schema]> SELECT STATEMENT_SCHEMA,STATEMENT_TEXT,RESULT_BLOCKS_COUNT,RESULT_BLOCKS_SIZE,RESULT_BLOCKS_SIZE_USED FROM QUERY_CACHE_INFO ORDER BY RESULT_BLOCKS_SIZE DESC;
+------------------+------------------------------+---------------------+--------------------+-------------------------+
| STATEMENT_SCHEMA | STATEMENT_TEXT               | RESULT_BLOCKS_COUNT | RESULT_BLOCKS_SIZE | RESULT_BLOCKS_SIZE_USED |
+------------------+------------------------------+---------------------+--------------------+-------------------------+
| test             | select    *  from  employees |                   2 |             261880 |                  261880 |
| test             | select * from  employees     |                   2 |             261880 |                  261880 |
| test             | select    * from  employees  |                   2 |             261880 |                  261880 |
| test             | select * from employees      |                   1 |             261816 |                  261816 |
| test             | SELECT * FROM employees      |                   1 |             261816 |                  261816 |
| test             | SELECT * from employees      |                   1 |             261816 |                  261816 |
| test             | select * FROM employees      |                   1 |             261816 |                  261816 |
+------------------+------------------------------+---------------------+--------------------+-------------------------+
7 rows in set (0.001 sec)
```
:::note warn
結果からわかるようにクエリキャッシュでは、同じデータを取得するSELECTでもバイト単位で一致しないと別のクエリとみなされてしまうので、注意してください。
:::

重要な部分だけ表にまとめます。それ以外の項目については、[ドキュメント](https://mariadb.com/kb/en/information-schema-query_cache_info-table/)を参照してください。
| 項目 | 内容 |
|:-----------|:------------|
|STATEMENT_SCHEMA|使用されたデータベース|
|STATEMENT_TEXT|ステートメントテキスト（クエリテキスト）|
|RESULT_BLOCKS_COUNT|結果ブロックの数|
|RESULT_BLOCKS_SIZE|結果ブロックのサイズ（バイト単位）|
|RESULT_BLOCKS_SIZE_USED|実際に使用されたブロックのサイズ|

### クエリキャッシュのキャッシュが削除されるタイミングについて

クエリキャッシュのキャッシュが削除されるタイミングの詳細を書いておきます。（大切なので）
削除されてしまうのは以下のタイミングです。

* DB再起動,RESET QUERY CACHEコマンドを実行したとき
* `query_cache_size`不足
* データ更新による削除

その中でも、データ更新による削除は内容が明確ではありません。データ更新とは、どの程度のデータ更新かということですが、以下のような感じになっています。
まず、以下のクエリと結果がキャッシュされているとします。emplayeesのidが1のものがキャッシュされているとします。
```sql
SELECT * FROM employees WHERE id =1
```
すると、以下の操作が行われたときに上記のキャッシュが削除されます。
* employeesテーブルにUPDATE,INSERT,DELETEが実行されたとき
* これは、id=1以外の結果が更新された場合でも削除されてしまいます。
* テーブル結合していた場合も同様です。結合先のテーブルが更新されれば、結果に関係なくともキャッシュは削除されてしまいます。


**※データ更新が多い環境だとほぼほぼキャッシュできない事がわかります。**

## さいごに

今回クエリキャッシュについてまとめました。
静的なサイトだと効果は絶大だと思いますので、バージョン低いmysqlを使用している方は是非クエリキャッシュの使用を検討してみてください。

