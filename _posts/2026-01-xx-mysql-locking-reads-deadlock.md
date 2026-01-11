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
