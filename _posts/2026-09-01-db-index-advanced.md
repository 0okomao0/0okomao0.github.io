---
title: 振り返り：「複合インデックス」
author: koma77
date: 2026-09-01 00:00:00 +0900
categories: [DB]
tags: [DB, 振り返り]
render_with_liquid: false
---

# 目的
[前回の記事](/posts/db-index/)では単一インデックスしか試してなかったので、せっかくなら複合インデックスもやってみたいな〜という思いつき

## 検証環境
前回と同様の環境・テーブルを使うので、前提は省略

# 複合インデックス
そもそも複合インデックスは「2つ以上の複数のカラムを組み合わせて作成するインデックス」

前回の本の例えから派生して、電話帳で例えると、  
単一インデックスは、単に名前（姓＋名とか）だけで索引を作成する  
複合インデックスは、まず姓で分類した後、名を並べて索引を作成する  
というような違いがある

## 単一インデックス2本と複合インデックスを比べる

でも複合インデックスって別に単一インデックス2本で良くない？という話なのでちょっと比べてみる

### 読み取り性能を比べる
カテゴリが42で、日時が指定した範囲にあるレコードを探してみる

まず、インデックスなしの`records`テーブルで実行する

```sql
mysql> EXPLAIN ANALYZE
    -> SELECT *
    -> FROM records
    -> WHERE category_id = 42
    ->   AND created_at >= '2020-04-01 00:00:00'
    ->   AND created_at <  '2020-04-02 00:00:00';
+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| EXPLAIN                                                                                                                                                                                                                                                                                                                                 |
+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| -> Filter: ((records.category_id = 42) and (records.created_at >= TIMESTAMP'2020-04-01 00:00:00') and (records.created_at < TIMESTAMP'2020-04-02 00:00:00'))  (cost=1.08e+6 rows=110070) (actual time=289..4928 rows=86 loops=1)
    -> Table scan on records  (cost=1.08e+6 rows=9.91e+6) (actual time=1.65..4593 rows=10e+6 loops=1)
 |
+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (4.93 sec)
```

次に、`category_id`と`created_at`それぞれに単一インデックスを作ったテーブルで、まったく同じ検索を実行する

```sql
CREATE TABLE records_with_index LIKE records;

INSERT INTO records_with_index 
SELECT * FROM records;

CREATE INDEX records_category_id_idx ON records_with_index (category_id);

CREATE INDEX records_created_at_idx ON records_with_index (created_at);
```

```sql
mysql> EXPLAIN ANALYZE
    -> SELECT *
    -> FROM records_with_index
    -> WHERE category_id = 42
    ->   AND created_at >= '2020-04-01 00:00:00'
    ->   AND created_at <  '2020-04-02 00:00:00';
+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| EXPLAIN                                                                                                                                                                                                                                                                                                                                                                            |
+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| -> Filter: ((records_with_index.created_at >= TIMESTAMP'2020-04-01 00:00:00') and (records_with_index.created_at < TIMESTAMP'2020-04-02 00:00:00'))  (cost=10018 rows=179) (actual time=50.8..2257 rows=86 loops=1)
    -> Index lookup on records_with_index using records_category_id_idx (category_id=42)  (cost=10018 rows=10000) (actual time=0.26..2252 rows=10000 loops=1)
 |
+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (2.26 sec)
```

最後に、同じデータで`(category_id, created_at)`の複合インデックスを作り、同じ検索を実行する

```sql
CREATE TABLE records_with_composite_index LIKE records;

INSERT INTO records_with_composite_index
SELECT * FROM records;

CREATE INDEX records_category_created_idx ON records_with_composite_index (category_id, created_at);
```

```sql
mysql> EXPLAIN ANALYZE
    -> SELECT *
    -> FROM records_with_composite_index
    -> WHERE category_id = 42
    ->   AND created_at >= '2020-04-01 00:00:00'
    ->   AND created_at <  '2020-04-02 00:00:00';
+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| EXPLAIN                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| -> Index range scan on records_with_composite_index using records_category_created_idx over (category_id = 42 AND '2020-04-01 00:00:00' <= created_at < '2020-04-02 00:00:00'), with index condition: ((records_with_composite_index.category_id = 42) and (records_with_composite_index.created_at >= TIMESTAMP'2020-04-01 00:00:00') and (records_with_composite_index.created_at < TIMESTAMP'2020-04-02 00:00:00'))  (cost=104 rows=86) (actual time=0.929..27.5 rows=86 loops=1)
 |
+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.03 sec)
```

今回の結果はこんな感じ

| インデックス構成 | 最初の1行が返るまで | 全86行が返るまで |
| --- | ---: | ---: |
| インデックスなし | 約0.289秒（289ms） | 約4.928秒（4928ms） |
| 単一インデックス2本 | 約0.051秒（50.8ms） | 約2.257秒（2257ms） |
| 複合インデックス1本 | 約0.001秒（0.929ms） | 約0.028秒（27.5ms） |

単一インデックスの方は`created_at`のインデックス使用していないので単純比較は出来ないけど、複合インデックスが速いのはわかる

### インデックスが使う容量を比べる
```sql
mysql> SELECT
    ->   table_name,
    ->   ROUND(data_length / 1024 / 1024, 2) AS data_size_mb,
    ->   ROUND(index_length / 1024 / 1024, 2) AS index_size_mb
    -> FROM information_schema.tables
    -> WHERE table_schema = DATABASE()
    ->   AND table_name IN (
    ->     'records',
    ->     'records_with_index',
    ->     'records_with_composite_index'
    ->   )
    -> ORDER BY table_name;
+------------------------------+--------------+---------------+
| TABLE_NAME                   | data_size_mb | index_size_mb |
+------------------------------+--------------+---------------+
| records                      |      1462.00 |          0.00 |
| records_with_composite_index |      1462.00 |        214.80 |
| records_with_index           |      1462.00 |        341.41 |
+------------------------------+--------------+---------------+
3 rows in set (0.00 sec)
```

今回の結果はこんな感じ

| テーブル | インデックス構成 | データ容量 | インデックス容量 |
| --- | --- | ---: | ---: |
| `records` | なし | 1462.00 MB | 0.00 MB |
| `records_with_index` | `category_id`と`created_at`の単一インデックス2本 | 1462.00 MB | 341.41 MB |
| `records_with_composite_index` | `(category_id, created_at)`の複合インデックス1本 | 1462.00 MB | 214.80 MB |


### 書き込み性能を比べる

前回と同じく、すでに1000万行あるテーブルに、さらに同じ1000万行を追加して所要時間を比べてみる

```sql
mysql> SET @started_at = NOW(6);
Query OK, 0 rows affected (0.00 sec)

mysql> INSERT INTO records (id, category_id, created_at, payload)
    -> SELECT
    ->   seq.n + 10000000,
    ->   MOD(seq.n, 1000) + 1,
    ->   TIMESTAMP('2020-01-01 00:00:00') + INTERVAL (seq.n + 10000000) SECOND,
    ->   REPEAT('x', 100)
    -> FROM seq;
Query OK, 10000000 rows affected (51.94 sec)
Records: 10000000  Duplicates: 0  Warnings: 0

mysql> SELECT TIMESTAMPDIFF(MICROSECOND, @started_at, NOW(6)) / 1000000 AS elapsed_seconds;
+-----------------+
| elapsed_seconds |
+-----------------+
|         51.9436 |
+-----------------+
1 row in set (0.00 sec)
```

同じINSERTを単一インデックス2本のテーブルに対して実行する

```sql
mysql> SET @started_at = NOW(6);
Query OK, 0 rows affected (0.00 sec)

mysql> INSERT INTO records_with_index (id, category_id, created_at, payload)
    -> SELECT
    ->   seq.n + 10000000,
    ->   MOD(seq.n, 1000) + 1,
    ->   TIMESTAMP('2020-01-01 00:00:00') + INTERVAL (seq.n + 10000000) SECOND,
    ->   REPEAT('x', 100)
    -> FROM seq;
Query OK, 10000000 rows affected (1 min 56.81 sec)
Records: 10000000  Duplicates: 0  Warnings: 0

mysql> SELECT TIMESTAMPDIFF(MICROSECOND, @started_at, NOW(6)) / 1000000 AS elapsed_seconds;
+-----------------+
| elapsed_seconds |
+-----------------+
|        116.8122 |
+-----------------+
1 row in set (0.00 sec)
```

複合インデックスのテーブルでも同じデータを追加する

```sql
mysql> SET @started_at = NOW(6);
Query OK, 0 rows affected (0.00 sec)

mysql> INSERT INTO records_with_composite_index (id, category_id, created_at, payload)
    -> SELECT
    ->   seq.n + 10000000,
    ->   MOD(seq.n, 1000) + 1,
    ->   TIMESTAMP('2020-01-01 00:00:00') + INTERVAL (seq.n + 10000000) SECOND,
    ->   REPEAT('x', 100)
    -> FROM seq;


Query OK, 10000000 rows affected (16 min 58.58 sec)
Records: 10000000  Duplicates: 0  Warnings: 0

mysql> SELECT TIMESTAMPDIFF(MICROSECOND, @started_at, NOW(6)) / 1000000 AS elapsed_seconds;
+-----------------+
| elapsed_seconds |
+-----------------+
|       1018.5777 |
+-----------------+
1 row in set (0.00 sec)
```

今回の結果はこんな感じ


| インデックス構成 | 追加行数 | INSERT所要時間 |
| --- | ---: | ---: |
| インデックスなし | 1000万行 | 約51.94秒 |
| 単一インデックス2本 | 1000万行 | 約116.81秒 |
| 複合インデックス1本 | 1000万行 | 約1018.58秒（約16分59秒） |

恐ろしいくらい複合インデックスの書き込みに時間がかかってる

ひとつの要因として、インデックスの並び順とINSERTする値の順序が考えられる  
今回のデータでは`created_at`が増加していく一方、`category_id`は1〜1000を繰り返す  
単一インデックスの`created_at`は末尾方向への追加になりやすいが、複合インデックス`(category_id, created_at)`では先頭の`category_id`が繰り返し切り替わるため、複数のキー範囲に交互に書き込む動きになる


#### 補足：`category_id`を同じ値にして書き込み時間を比べる

あまりにも書き込み時間に差が出来てしまったので、同じ1000万行を`category_id`だけ固定値`1000`にして実施してみる  
なお、テーブルは書き込み性能を調べる前の状態（records/ records_with_index/ records_with_composite_index 全て同じ1000万行を保有）に戻してから実施した

```sql
mysql> SET @started_at = NOW(6);
Query OK, 0 rows affected (0.00 sec)

mysql> INSERT INTO records (id, category_id, created_at, payload)
    -> SELECT
    ->   records.id + 10000000,
    ->   1000,
    ->   TIMESTAMP('2020-01-01 00:00:00') + INTERVAL (records.id + 10000000) SECOND,
    ->   REPEAT('x', 100)
    -> FROM records;
Query OK, 10000000 rows affected (57.85 sec)
Records: 10000000  Duplicates: 0  Warnings: 0

mysql> SELECT TIMESTAMPDIFF(MICROSECOND, @started_at, NOW(6)) / 1000000 AS elapsed_seconds;
+-----------------+
| elapsed_seconds |
+-----------------+
|         57.8571 |
+-----------------+
1 row in set (0.00 sec)
```

```sql
mysql> SET @started_at = NOW(6);
Query OK, 0 rows affected (0.00 sec)

mysql> INSERT INTO records_with_index (id, category_id, created_at, payload)
    -> SELECT
    ->   records.id + 10000000,
    ->   1000,
    ->   TIMESTAMP('2020-01-01 00:00:00') + INTERVAL (records.id + 10000000) SECOND,
    ->   REPEAT('x', 100)
    -> FROM records;
Query OK, 10000000 rows affected (1 min 30.68 sec)
Records: 10000000  Duplicates: 0  Warnings: 0

mysql> SELECT TIMESTAMPDIFF(MICROSECOND, @started_at, NOW(6)) / 1000000 AS elapsed_seconds;
+-----------------+
| elapsed_seconds |
+-----------------+
|         90.6818 |
+-----------------+
1 row in set (0.00 sec)
```

```sql
mysql> SET @started_at = NOW(6);
Query OK, 0 rows affected (0.00 sec)

mysql> INSERT INTO records_with_composite_index (id, category_id, created_at, payload)
    -> SELECT
    ->   records.id + 10000000,
    ->   1000,
    ->   TIMESTAMP('2020-01-01 00:00:00') + INTERVAL (records.id + 10000000) SECOND,
    ->   REPEAT('x', 100)
    -> FROM records;
Query OK, 10000000 rows affected (1 min 23.91 sec)
Records: 10000000  Duplicates: 0  Warnings: 0

mysql> SELECT TIMESTAMPDIFF(MICROSECOND, @started_at, NOW(6)) / 1000000 AS elapsed_seconds;
+-----------------+
| elapsed_seconds |
+-----------------+
|         83.9069 |
+-----------------+
1 row in set (0.00 sec)
```

今回の結果はこんな感じ

| インデックス構成 | 追加行数 | INSERT所要時間 |
| --- | ---: | ---: |
| インデックスなし | 1000万行 | 約57.86秒 |
| 単一インデックス2本 | 1000万行 | 約90.68秒 |
| 複合インデックス1本 | 1000万行 | 約83.90秒 |

流石にインデックスなしよりは時間がかかっているが、前回みたいな16分とかにはならなかった  
インデックスを貼っている場合は読み取り（検索）条件だけでなく、書き込むデータにも配慮してあげないといけないことを実感した

### 注意点1：カラムの順序が重要

複合インデックス`(category_id, created_at)`は、まず`category_id`、その値の中で`created_at`の順に並んでいる  
そのため、先頭のカラムから条件を指定する検索には適している

```sql
-- 先頭カラムだけを使う検索
SELECT *
FROM records_with_composite_index
WHERE category_id = 42;

-- 先頭カラムと2番目のカラムを使う検索
SELECT *
FROM records_with_composite_index
WHERE category_id = 42
  AND created_at >= '2020-04-01 00:00:00'
  AND created_at <  '2020-04-02 00:00:00';
```

でも、2番目のカラムだけ（または2番目以降のカラム）を使用した検索条件には使えない

```sql
-- category_idを指定していない
SELECT *
FROM records_with_composite_index
WHERE created_at >= '2020-04-01 00:00:00'
  AND created_at <  '2020-04-02 00:00:00';
```
