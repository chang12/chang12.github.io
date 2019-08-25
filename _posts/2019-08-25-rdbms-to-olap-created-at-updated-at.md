---
layout: post
title: RDBMS to OLAP 에 대한 생각 - created_at, updated_at 컬럼
tags:
  - Data Engineering Best Practices
---

데이터 엔지니어의 중요한 업무 중 하나는 각종 소스의 데이터를 적절한 [OLAP 데이터베이스로](https://en.wikipedia.org/wiki/Online_analytical_processing) 로 옮기는 것입니다. 이때 비중이 큰 소스 중 하나가 백엔드 시스템이 사용하는 RDBMS 입니다. 실제로, AWS Aurora MySQL 의 데이터를 Google Cloud 의 BigQuery 로 주기적으로 옮기는 데이터 파이프라인을 만드는데 많은 시간을 썼었습니다. 한번 작업을 마치고 나니 상당히 전형적인 작업이라는 생각이 들었습니다. 그래서 Best Practices 에 대해 생각해보고 있습니다.

## RDBMS 에 대한 가정 : `created_at`, `updated_at` 컬럼

**RDBMS 의 모든 테이블에 TIMESTAMP 타입의 `created_at`, `updated_at` 컬럼이 존재하고, 두 컬럼 각각에 대한 index 가 존재한다고 가정합니다.** 이는 곧 백엔드 시스템이 사용하는 RDBMS 이 지켜줬으면 하는 희망 사항입니다.

- `created_at` : record 가 insert 된 시각. insert 때 값이 정해지고 이후 불변합니다.
- `updated_at` : record 가 마지막으로 update 된 시각. insert 직후에는 `created_at` 컬럼의 값과 동일하고, 이후 update 때마다 값이 갱신됩니다.

이를 통해 RDBMS 의 모든 테이블에 대해 몇 가지 SQL 이 효율적으로 이뤄질 수 있다고 가정할 수 있습니다.

- 특정 기간동안 업데이트 된 records 를 `select`
- 특정 기간동안 최초 생성된 records 를 `select`

예를 들어 (AWS Aurora MySQL 이 제공하는 가장 최신 버전인) MySQL 5.7 버전에서는 [default 와 on update 구문을 활용하여](https://dev.mysql.com/doc/refman/5.7/en/timestamp-initialization.html), DBMS 단에서 DDL 만으로 위의 조건을 선언할 수 있습니다.

```sql
create table t (
  id int not null auto_increment,
  name varchar(8),
  created_at timestamp not null default current_timestamp,
  updated_at timestamp not null default current_timestamp on update current_timestamp,

  primary key (id),
  index (created_at),
  index (updated_at)  
);
```

`created_at`, `updated_at` 값을 명시하지 않아도, 알아서 값이 채워집니다.

```sql
insert into t (name) values ('A');
insert into t (name) values ('B');
insert into t (name) values ('C');

-- id	name	created_at	updated_at
-- 1	A	2019-08-25 12:36:04	2019-08-25 12:36:04
-- 2	B	2019-08-25 12:36:07	2019-08-25 12:36:07
-- 3	C	2019-08-25 12:36:09	2019-08-25 12:36:09
```

update 때 `updated_at` 값이 알아서 갱신됩니다.

```sql
update t set name = 'AA' where id = 1;

-- id	name	created_at	updated_at
-- 1	AA	2019-08-25 12:36:04	2019-08-25 12:36:49
-- 2	B	2019-08-25 12:36:07	2019-08-25 12:36:07
-- 3	C	2019-08-25 12:36:09	2019-08-25 12:36:09
```

## OLAP 데이터베이스에 대한 가정 : `DATE` Partitioned 테이블

**OLAP 데이터베이스는 `DATE` 타입 컬럼에 대한 Partitioning 을 제공한다고 가정합니다.** 그러면 `DATE` Partition 단위로 테이블의 데이터를 교체할 수 있습니다.

예를 들어 Google Cloud 의 BigQuery 는 [DATE, TIMESTAMP 타입의 컬럼에 대한 Partitioning](https://cloud.google.com/bigquery/docs/partitioned-tables#partitioned_tables) 을 제공하고, [Partition 에 데이터를 append 하거나 overwrite](https://cloud.google.com/bigquery/docs/managing-partitioned-table-data#append-overwrite) 할 수 있는 api 를 제공합니다.

> BigQuery also allows partitioned tables. Partitioned tables allow you to bind the partitioning scheme to a specific **TIMESTAMP or DATE** column. Data written to a partitioned table is automatically delivered to the appropriate partition based on the date value (expressed in UTC) in the partitioning column.

> You can **overwrite** partitioned table data using a load or query operation. You can **append** additional data to an existing partitioned table by performing a load-append operation or by appending query results.

OLAP 데이터베이스로 많이 사용되는 오픈소스 프로젝트인 Presto 의 경우, Hive Connector 를 사용할 수 있고, [Hive 의 DDL 에서 Partitioning 을 제공합니다.](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+DDL#LanguageManualDDL-PartitionedTables) Partitioning 의 자유도가 더 높아서, 예를 들어 1일이 아니라 1시간 단위로도 Partition 을 나눌 수 있습니다.

## RDBMS 에서 OLAP 로 데이터 복사

앞서 가정한 내용들을 바탕으로 RDBMS 의 임의의 테이블의 데이터를 OLAP 데이터베이스로 주기적으로 복사하는 작업을 일반화 시킬 수 있습니다.

1. `created_at` 기준으로, RDBMS 테이블이 생성된 날짜 ~ 데이터 복사 시작일 1일 전 날짜까지 `DATE` Partition 복사.

    ```sql
    -- 예를 들어 2019-08-24 Partition 이라면,
    
    select 
      * 
    from
      t 
    where
      created_at >= '2019-08-24 00:00:00' 
      and created_at < '2019-08-25 00:00:00'
    ```

2. `UPDATED_AT_CKPT` (임의로 지은 이름. 테이블마다 관리) 값을 데이터 복사 시작일 00:00:00 로 초기화.
3. `UPDATED_AT_CKPT` ~ 현재 시각 사이에 update 된 records 들의 `created_at` 기준 `DATE` 값의 distinct 한 목록 추출.

    ```sql
    select
      distinct(date(created_at))
    from
      t
    where
      updated_at >= {UPDATED_AT_CKPT}
      and updated_at < current_timestamp
    ```
4. 추출한 날짜 목록의 OLAP 데이터베이스 `DATE` Partition 를 모두 교체.
5. `UPDATED_AT_CKPT` 값을 3 에서 쓴 current_timestamp 값으로 갱신.
6. 3 ~ 5 과정을 적절한 주기로 반복.

### `DATE` Partition 을 `created_at` 기준으로 나누는 이유

`updated_at` 기준으로 할 경우 record 의 update 시점과 OLAP 데이터베이스로 데이터를 복사하는 프로그램의 실행 시점에 따라서, 여러 `DATE` Partition 에 걸쳐 record 의 중복이 발생할 수 있습니다. 예를 들어,

- RDBMS 에 2019-08-23 중 어떤 record (= R) 이 insert.
- R 이 포함된 2019-08-23 Partition 이 OLAP 데이터베이스로 복사.
- RDBMS 에서 2019-08-24 중 R 의 어떤 컬럼을 update.
- update 된 R 이 포함된 2019-08-24 Partition 이 OLAP 데이터베이스로 복사.
- R 은 2019-08-23 과 2019-08-24 2개 Partition 에 중복.

따라서 RDBMS 의 record 에 대해 불변하는 `created_at` 을 기준으로 OLAP 데이터베이스의 Partition 을 나눕니다.

## 레퍼런스

- [MySQL 5.7 문서 중 11.3.5 Automatic Initialization and Updating for TIMESTAMP. and DATETIME](https://dev.mysql.com/doc/refman/5.7/en/timestamp-initialization.html)
- [Apache Hive - LanguageManual DDL - Partitioned Tables](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+DDL#LanguageManualDDL-PartitionedTables)
