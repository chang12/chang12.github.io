---
layout: post
title: "BigQuery 에서 slot utilization 을 집계하는 SQL"
tags: []
---

bigquery 에서 compute pricing 에 on-demand pricing 과 capacity pricing 으로 2가지가 있다. capacity pricing 은 slot 기반의 pricing 이다. 원하는 slot 개수만큼 임대하여 사용하고, 그만큼 비용을 지불한다. 그러니 임대한 slot 을 효과적으로 쓰고 있는지 monitoring 할 필요가 있다. 효과적으로 쓰고 있는지 = 실제 사용한 slot 개수 / 임대한 slot 개수 = slot utilization 으로 볼 수 있겠다.

[Example: See total slot usage per minute](https://cloud.google.com/bigquery/docs/information-schema-reservation-timeline#example_see_total_slot_usage_per_minute) 문서에서 이러한 slot utilization 을 집계하는 sql 을 알려준다. 그런데 이 sql 을 그대로 가져다가 실행 했을 때 `slot_assigned` 가 0 으로 나온다.  reservation 의 baseline slots 은 0 으로 하고 / max size 를 조절하며 autoscale 하고 있기 때문이다. 저 sql 에서는 autoscale 로 조절되는 slot 개수를 고려하지 않는다.

`JOBS_TIMELINE` view 는 [매 second 마다](https://cloud.google.com/bigquery/docs/information-schema-jobs-timeline#schema) 집계 되고, `RESERVATION_TIMELINE` view 는 [매 minute 마다](https://cloud.google.com/bigquery/docs/information-schema-reservation-timeline) 집계 된다. 그러니 매 minute 마다 집계하는 걸로 맞춘다.

```sql
with

job_timeline_by_minute as (

select
  project_id,
  reservation_id,
  timestamp_trunc(period_start, minute) as period_start_truncated_to_minute,

  sum(period_slot_ms) as period_slot_ms,
from
  <project-id>.`region-us`.INFORMATION_SCHEMA.JOBS_TIMELINE
where
  -- Avoid duplicate byte counting in parent and children jobs.
  statement_type != 'SCRIPT'
  or statement_type is null
group by
  project_id
  , reservation_id
  , period_start_truncated_to_minute

)

select
  date(reservation_timeline_by_minute.period_start) as _date_kst,
  reservation_timeline_by_minute.project_id,
  reservation_timeline_by_minute.reservation_id,
  reservation_timeline_by_minute.period_start,
  reservation_timeline_by_minute.autoscale.current_slots as slots,
  job_timeline_by_minute.period_slot_ms / (60 * 1000) as slots_used,
from
  <project-id>.`region-us`.INFORMATION_SCHEMA.RESERVATION_TIMELINE as reservation_timeline_by_minute
left join job_timeline_by_minute on reservation_timeline_by_minute.project_id = job_timeline_by_minute.project_id
  and reservation_timeline_by_minute.reservation_id = job_timeline_by_minute.reservation_id
  and reservation_timeline_by_minute.period_start = job_timeline_by_minute.period_start_truncated_to_minute

```