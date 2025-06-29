---
layout: post
title: ""
tags: []
---

bigquery 에서 sql 작성할 때, group by 로 어떤 expression 의 최초 or 마지막 값을 취하고 싶을 때가 있다. e.g. wau 의 마지막 접속 device 를 얻고 싶다던지.

### row_number

그럴때, 처음에는 `row_number` 를 사용했다.

```sql
select
  ...
  device,
from (
  select
    ...
    device,
    row_number() over (partition by user_id order by ts) as rn,
  from
    event
)
where
  rn = 1
```

최초만 따지면 되는데, 모든 row 에 다 `row_number` 를 매겨야 한다는 게 불만족스러웠고, select 문 한겹으로 안되고 두겹으로 해야하는 것도 마음에 들지 않았다.

### qualify

[2022-02 에 `QUALIFY` clause 가 나오면서](https://cloud.google.com/bigquery/docs/release-notes#February_14_2022), 두겹으로 해야하는 불만족은 해소 되었다.

```sql
select
  ...
  device,
from
  event
qualify
  row_number() over (partition by user_id order by ts) = 1
```

### array_agg

그러던 중, 2022-03-05 에 bigquery 문서를 읽고, `array_agg` 를 쓰는 practice 에 대해 알게 되었다. 해당 문서에서는 `row_number` 를 쓰는 대신 `array_agg` 를 쓰는 게 best practice 라고 소개한다.

> Use aggregate analytic function to obtain the latest record
> 
> **Best practice**: To obtain the latest record, use the `ARRAY_AGG()` aggregate analytic function.
> 
> Using the `ARRAY_AGG()` aggregate analytic function instead of using numbering functions, such as `RANK()` or `ROW_NUMBER()`, allows a query to run more efficiently because the `ORDER BY` clause is allowed to drop everything except the top record on each `GROUP BY` clause. For example,
```sql
SELECT
  event.*
FROM (
  SELECT id, ARRAY_AGG(
    t ORDER BY t.created_at DESC LIMIT 1
  )[OFFSET(0)] event
  FROM
    `dataset.table` t
  GROUP BY
    id
)
```

겸사겸사, `from` 절에 적은 table 을 `select` 절에서 struct 으로 다룰 수 있다는 것도 배웠다.

해당 내용은 원래 bigquery [Optimize query computation](https://cloud.google.com/bigquery/docs/best-practices-performance-compute) 문서에 있었다. 그런데 글을 작성하는 시점에 다시 들어가 보니 내용이 없어졌다. [wayback machine](https://web.archive.org/web/20220501000000*/https://cloud.google.com/bigquery/docs/best-practices-performance-compute) 으로 확인 해보니, [2022-07-03 snapshot](https://web.archive.org/web/20220703045100/https://cloud.google.com/bigquery/docs/best-practices-performance-compute#use_aggregate_analytic_function_to_obtain_the_latest_record) 까지는 있는데, [2022-08-29 snapshot](https://web.archive.org/web/20220829114913/https://cloud.google.com/bigquery/docs/best-practices-performance-compute) 부터 없어졌다.   

없어진 이유가 무엇일까? 하고 [BigQuery release notes](https://cloud.google.com/bigquery/docs/release-notes) 에서 2022-07 ~ 2022-08 의 내용을 확인 해봤으나, 딱히 짐작가는 바가 없다.

이 문단을 적으면서 문득, 이렇게 유용하게 잘 쓰는 `array_agg` 가 언제부터 있었을까? 궁금해져서 찾아보니, 적어도 2016-12-22 부터 존재한 오래된 function 이었다. [2016-12-22 자 release note](https://cloud.google.com/bigquery/docs/release-notes#December_22_2016) 에 아래 내용이 있다.

> Standard SQL now supports ORDER BY and LIMIT clauses within ARRAY_AGG, ARRAY_CONCAT_AGG, and STRING_AGG.

### min_by, max_by

그렇게 `array_agg` 를 잘 활용하고 있던 중, 2024-07-09 에 `min_by`, `max_by` function 이 있다는 걸 알게 되었다.

원래 이렇게 하던 걸,
```sql
select
  id,

  array_agg(xxx order by created_at limit 1)[safe_offset(0)] as record,
from
  xxx
group by
  id
```
더 간결하게 이렇게 할 수 있는.
```sql
select
  id,

  min_by(xxx, created_at) as record,
from
  xxx
group by
  id
```

[MAX_BY](https://cloud.google.com/bigquery/docs/reference/standard-sql/aggregate_functions#max_by), [MIN_BY](https://cloud.google.com/bigquery/docs/reference/standard-sql/aggregate_functions#min_by) 문서를 보면 `Synonym for ANY_VALUE(x HAVING MAX/MIN y).` 라고 한다. bigquery release notes 를 보니, [2023-02-06 에](https://cloud.google.com/bigquery/docs/release-notes#February_06_2023) `any_value` function 에 having max/min clause 가 preview 로 추가 되었고, [2023-08-08 에](https://cloud.google.com/bigquery/docs/release-notes#August_08_2023) generally available 되면서, 동시에 `max_by` 와 `min_by` 도 추가 되었다.   

### min_by, max_by 의 함정

그래서 `min_by`, `max_by` 를 알게된 후로 query 를 작성할 때는 `array_agg` 대신 `min_by`, `max_by` 를 쓰기 시작했다. 그리고 틈날때 마다, `array_agg` 를 쓰던 기존 query 들을 -> `min_by`, `max_by` 를 쓰도록 수정했다.

그러던 중 2025-02-13 에 daily batch 의 특정 query job 의 실행 시간이 어제 대비 급격히 증가하는 것을 관찰했다. 직전 실행 이후 어떤 수정이 있었는지 commit history 를 확인 해보니, `array_agg` 쓰던 부분을 `min_by` 로 바꾼 것 뿐 이었다.

query job 실행 시간이 달라질 만한 수정은 아니라고 생각하여 의아했는데, 확인 해보니, `array_agg` 를 쓸 때와, `min_by` 를 쓸 때, bigquery console 에서 보여주는 **estimated bytes processed** 가 다르다.

e.g.

```sql
select
  record.field1
from (
  select
    array_agg(event order by event_at limit 1)[safe_offset(0)] as record,
  from
    <some_event_table> as event
)
```

`array_agg` 를 쓸 때는 **106.98 GB** 인데,

```sql
select
  record.field1
from (
  select
    min_by(event, event_at) as record,
  from
    <some_event_table> as event
)
```

`min_by` 를 쓸 때는 **1.11 TB** 로 커진다.

```sql
select
  record.field1
from (
  select
    any_value(event having min event_at) as record,
  from
    <some_event_table> as event
)
```

문서에서 `Synonym for ANY_VALUE(x HAVING MAX/MIN y).` 라고 했던 것 처럼, `any_value` 쓸 때도 `min_by` 와 동일하게 **1.11 TB** 로 커진다. 

<some_event_table> 에서 결과적으로 `event_at` 과 `field1` 만 사용하는 데,
- `array_agg` 쓰는 query 에서는 그게 인지 되어 `event_at` 과 `field1` 만큼만 estimated bytes processed 로 잡히고, 
- `min_by` 와 `any_value` 를 쓰는 query 에서는 인지가 안되어 <some_event_table> 전체가 estimated bytes processed 로 잡히는 것이다.

### execution details 에 차이가 있다.

`array_agg` 를 쓰는 query 의 execution details 은 아래와 같다. 처음부터 `$1:event_at, $2:field1 FROM event.test` 로 `event_at` 과 `field1` 만 취하고, 이후 step 을 진행하는 것을 볼 수 있다.  
```
[S00: Input]
$1:event_at, $2:field1 FROM event.test
$30 := SHARD_ARRAY_AGG($2 ORDER BY $1 ASC LIMIT 1)
$30 TO __stage00_output

[S01: Output]
$30 FROM __stage00_output
$10 := safe_array_at_offset($20, 0)
$20 := ROOT_ARRAY_AGG($30 ORDER BY  ASC LIMIT 1)
$10 TO __stage01_output
```

그에 비해 `min_by` 를 쓰는 query 의 execution details 은 아래와 같다. `event_at` 과 `field1` 만 필요하다는 것에 대한 인지 없이, `MAKE_STRUCT($1, $2, $3, $4, $5, 1)` 해버리는 걸 볼 수 있다.
```
[S00: Input]
$1:user_id, $2:created_at, $3:_date, $4:event_at, $5:field1 FROM event.test
$30 := SHARD_ANY_VALUE_HAVING($40)
$40 := MAKE_STRUCT($1, $2, $3, $4, $5, 1)
$30 TO __stage00_output

[S01: Output]
$30 FROM __stage00_output
$10 := STRUCT_FIELD_OP(4, $20)
$20 := ROOT_ANY_VALUE_HAVING($30)
$10
TO __stage01_output
```

더 찾아보니, 이렇게 전체 query 를 파악하여 필요한 column 만 취하여 사용하는 걸 **column pruning** 이라 부른다고. 
