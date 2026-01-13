---
layout: post
title: "mysql locking reads, deadlock, 중복 방지"
tags: []
---

mysql 8.0 은 [locking reads](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking-reads.html) 를 지원한다.

locking reads 때 어떤 lock 을 사용하는 지는, [isolation level 에 따라 달라진다.](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html)

read committed 일때는 record lock 만 사용한다.

> For locking reads (SELECT with FOR UPDATE or FOR SHARE), UPDATE statements, and DELETE statements, InnoDB **`locks only index records, not the gaps before them,`** and thus permits the free insertion of new records next to locked records.

repeatable read 이고 & `unique index with a unique search condition` 이 아니고 `range-type search condition` 인 경우, gap locks or next-key locks 을 사용한다.

> For other search conditions, InnoDB locks the index range scanned, **`using gap locks or next-key locks`** to block insertions by other sessions into the gaps covered by the range.

[next-key lock = record lock + gap lock](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html#innodb-next-key-locks) 이다.

> A next-key lock is a combination of a record lock on the index record and a gap lock on the gap before the index record.

[2개 이상의 transaction 이 같은 gap lock 을 획득할 수 있다.](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html#innodb-next-key-locks)

> Gap locks in InnoDB are “purely inhibitive”, which means that their only purpose is to prevent other transactions from inserting to the gap. Gap locks can co-exist. A gap lock taken by one transaction does not prevent another transaction from taking a gap lock on the same gap. There is no difference between shared and exclusive gap locks. They do not conflict with each other, and they perform the same function.

그러니 2개 이상의 transaction 이 같은 조건으로 `select ... for update` 하는 데는 문제가 없다.

하지만 각 transaction 내에서 이어서 같은 range 에 insert 하려는 경우, 얘기가 달라진다. insert 하기 위해 gap 에 대한 [insert intention lock](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html#innodb-insert-intention-locks) 을 획득 하려고 하나, 다른 transaction 이 같은 gap 에 대해 이미 gap lock 을 가지고 있으므로, deadlock 에 빠지게 된다.

[InnoDB Data Locking - Part 2 "Locks"](https://dev.mysql.com/blog-archive/innodb-data-locking-part-2-locks/) 에서 이러한 gap lock 과 insert intention lock 간의 priority 에 대해 더 자세하게 다루는 것 같다. 제대로 읽어보진 못했다.

이러한 deadlock 이 발생했을 때, 무한히 교착 상태에 빠지는 것은 아니고, [innodb 에서 적절히 일부 transaction 들을 rollback 시킨다.](https://dev.mysql.com/doc/refman/8.0/en/innodb-deadlock-detection.html)

> When deadlock detection is enabled (the default), InnoDB automatically detects transaction deadlocks and rolls back a transaction or transactions to break the deadlock. InnoDB tries to pick **`small transactions to roll back`**, where the size of a transaction is determined by the number of rows inserted, updated, or deleted.

local 에서 간단히 test 해서 재현 해볼 수 있다.

```bash
docker run --name mysql-deadlock-test \
  -e MYSQL_ALLOW_EMPTY_PASSWORD=yes \
  -e MYSQL_DATABASE=testdb \
  -p 3306:3306 \
  -d mysql:8.0
```

```bash
docker exec -it mysql-deadlock-test mysql testdb
```

```sql
create table t (id int primary key);
insert into t values (10), (20), (30);
```

terminal 2개를 열고 각각 접속해서, 

```bash
docker exec -it mysql-deadlock-test mysql testdb
```

아래 순서로 실행한다.

| 1                                           | 2                                                                                        |
|---------------------------------------------|------------------------------------------------------------------------------------------|
| `begin;`                                    |                                                                                          |
| `select * from t where id = 25 for update;` |                                                                                          |
|                                             | `begin;`                                                                                 |
|                                             | `select * from t where id = 25 for update;`                                              |
| `insert into t values (25);`                |                                                                                          |
| (멈춰 있음)                                     | `insert into t values (25);`                                                             |
| `Query OK, 1 row affected (3.85 sec)`       | `ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction` |
