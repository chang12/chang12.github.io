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
