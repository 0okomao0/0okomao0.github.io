---
title: 振り返り：「トランザクション」
author: koma77
date: 2026-09-01 00:00:00 +0900
categories: [DB]
tags: [DB, 振り返り]
render_with_liquid: false
---

# 目的
「トランザクション」を銀行口座の振込で説明し、実際のSQLで動きを確認してみる

# トランザクションとは

## 一言で言うなら
複数の処理をひとまとまりとして扱う仕組み  
よく言われるのは「銀行の振込」

銀行口座でAさんからBさんへ1,000円を振り込む場合、実際には次の2つの更新が必要になる

1. Aさんの口座から1,000円を引く
2. Bさんの口座に1,000円を足す

片方だけ成功してもう片方が失敗すると、お金が消えたり増えたりしてしまう  
両方成功したときだけ確定し、途中で問題が起きたら両方とも取り消すためにトランザクションを使う

## ACID特性

トランザクションが備える性質を、頭文字をとってACID特性と呼ぶ  
銀行振込に当てはめると次のようになる

| 特性 | 意味 | 振込の例 |
| --- | --- | --- |
| Atomicity（原子性） | すべて実行するか、すべて取り消す | 引き落としと入金の両方が成功するか、失敗時は両方を取り消す |
| Consistency（一貫性） | トランザクションの前後でデータの整合性を保つ | 残高をマイナスにしない、振込分だけ送金元から送金先へ移す |
| Isolation（独立性） | 同時に行われる処理が互いに不整合な状態を見ない | 別の振込処理が途中の更新だけを見て計算しない |
| Durability（永続性） | 確定した変更が保存される | `COMMIT`が成功した振込結果は、その後もデータベースに残る |

`ROLLBACK`による取り消しは主に原子性、`COMMIT`後も結果が残ることは永続性に関係する  
残高がマイナスにならないといった制約は、一貫性を保つためのルールとしてデータベースに設定できる

ただし、トランザクションを使えば業務上のルールがすべて自動で守られるわけではない  
送金元・送金先の更新件数を確認してから確定する処理は、アプリケーション側で正しく実装する必要がある。また、同時実行時の見え方は分離レベルなどの設定にも依存する

## 検証環境

前回と同じく、MySQL 8.4をDockerで起動する

```bash
docker run --rm --name transaction-mysql \
  --cpus=2 \
  --memory=4g \
  --memory-swap=4g \
  -e MYSQL_ROOT_PASSWORD=password \
  -p 127.0.0.1:3306:3306 \
  -d mysql:8.4
```

MySQLの起動完了を待ち、接続する

```bash
docker exec transaction-mysql mysqladmin ping -uroot -ppassword --wait=30
docker exec -it transaction-mysql mysql -uroot -ppassword
```

## 口座テーブルを作る

`InnoDB`はトランザクションに対応したストレージエンジン  
残高は小数を正確に扱える`DECIMAL`型にする

```sql
CREATE DATABASE transaction_lab;
USE transaction_lab;

CREATE TABLE accounts (
  account_id   INT PRIMARY KEY,
  owner_name   VARCHAR(50) NOT NULL,
  balance      DECIMAL(12, 2) NOT NULL,
  CHECK (balance >= 0)
) ENGINE=InnoDB;

INSERT INTO accounts (account_id, owner_name, balance)
VALUES
  (1, 'Aさん', 10000.00),
  (2, 'Bさん', 5000.00);

SELECT * FROM accounts ORDER BY account_id;
```

はじめの残高はAさんが10,000円、Bさんが5,000円

## 振込を実行する

AさんからBさんへ1,000円を振り込む  
最初にトランザクションを開始し、送金元から引き落とし、送金先へ入金した後、最後に`COMMIT`して確定する

以下は、1つのMySQL接続で順番に実行する。まずトランザクションを開始し、送金元から引き落とす

```sql
START TRANSACTION;

UPDATE accounts
SET balance = balance - 1000.00
WHERE account_id = 1
  AND balance >= 1000.00;
SELECT ROW_COUNT() AS debited_rows;
```

`debited_rows`が`1`なら引き落とし成功なので、次の入金へ進む  
`0`なら口座がないか残高不足のため、次のSQLを実行して中止する

```sql
ROLLBACK;
```

引き落としに成功した場合だけ、送金先へ入金する

```sql
UPDATE accounts
SET balance = balance + 1000.00
WHERE account_id = 2;
SELECT ROW_COUNT() AS credited_rows;
```

`credited_rows`が`1`なら入金成功。`0`なら送金先口座が存在しないため、`COMMIT`せずに`ROLLBACK`する

両方の更新件数が`1`だった場合だけ、トランザクションを確定する

```sql
COMMIT;
SELECT * FROM accounts ORDER BY account_id;
```

成功した場合の残高はAさんが9,000円、Bさんが6,000円になる

## 途中で失敗したら取り消す

次は、送金元から引き落とした後にエラーが起きた状況を試す  
`ROLLBACK`すると、トランザクション中の更新が取り消される

```sql
START TRANSACTION;

UPDATE accounts
SET balance = balance - 1000.00
WHERE account_id = 1
  AND balance >= 1000.00;

SELECT * FROM accounts ORDER BY account_id;

ROLLBACK;

SELECT * FROM accounts ORDER BY account_id;
```

`ROLLBACK`の前のSELECTではAさんの残高が一時的に減って見えるが、後のSELECTではトランザクション開始前の残高に戻る  
ここでは引き落としだけ行い、入金前に失敗した状況を再現している

## まとめ

銀行振込のように、複数の更新をまとめて成功または取り消しにしたいときにトランザクションを使う  
`START TRANSACTION`で開始し、すべて成功したら`COMMIT`、途中で失敗したら`ROLLBACK`する

その際に期待される性質がACID特性であり、途中までの更新だけが残ることや、整合性のない状態になることを防ぐ  
実際の処理では、更新件数やエラーを確認し、失敗時に確実に`ROLLBACK`する制御も必要
