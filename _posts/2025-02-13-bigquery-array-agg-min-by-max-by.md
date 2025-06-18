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

최초만 따지면 되는데, 모든 row 에 다 `row_number` 를 매겨야 한다는 게 불만족스러웠고, select 문 한번으로 안되고 두겹으로 해야하는 것도 마음에 들지 않았다.

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
