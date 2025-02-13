---
layout: post
title: ""
tags: []
---

bigquery 에서 sql 작성할 때, 여러 row 가 있는 상황에서 최초 or 마지막 값을 취하고 싶을 때가 있다. e.g. 유저를 앱으로 유입 시킨 channel 이 여러개 있는데, 그 중에서 최초로 유입 시킨 channel 에 attribution 을 주고 싶다던가.

그런 경우에, 처음에는 `row_number` 를 사용했다.

```sql
select
  ...
  channel,
from (
  select
    ...
    channel,
    row_number() over (partition by user_id order by ts) as rn,
  from
    event
)
where
  rn = 1
```

최초만 따지면 되는데, 모든 row 에 다 `row_number` 를 매겨야 한다는 게 불만족스러웠고, select 문 한번으로 안되고 두겹으로 해야하는 것도 마음에 들지 않았다.

[2022-02 에 `QUALIFY` clause 가 나오면서](https://cloud.google.com/bigquery/docs/release-notes#February_14_2022), 두겹으로 해야하는 불만족은 해소 되었다.

```sql
select
  ...
  channel,
from
  event
qualify
  row_number() over (partition by user_id order by ts) = 1
```
