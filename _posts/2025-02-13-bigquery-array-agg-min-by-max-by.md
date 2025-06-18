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
