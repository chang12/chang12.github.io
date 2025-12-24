---
layout: post
title: "bigquery -> mysql"
tags: []
---

# 배경

assign 된 epic 에서 진행 하려는 실험의 target users 를 구성하는 condition 들이, 보통의 실험들 보다 조금 까다로웠다.
1. 같은 목적의 이전 실험에 assign 되지 않았거나 or `control` 에 assign 되었음.  
2. 같은 목적의 이전 실험 기간 동안 어떤 screen 을 view 한 적 없음.

1번은 과거에도 요구된 바 있다. 이전 실험을 계속 RUNNING 으로 둔채로, 새로운 실험을 덧붙이는 식으로 구현 했었다.

```python
# 이전 실험을 assign 하고,
property1 = get_property('실험1', ...)
# assign 된 내용을 -> 새로운 실험 assign 때 고려.
property2 = get_property('실험2', context={property1, ...})
```

그런데 2번은 마땅한 방법이 없었다.

# 구현

그래서 bigquery 에서 해당 조건을 집계하여, mysql db 로 sync 하고, 그걸 사용하는 걸로 결정했다. 이전 실험은 종료하고, 그래서 `이전 실험 기간` 을 명확히 하고, bigquery 에서 query 한 result 를 mysql db 로 한번 sync 하고, 그걸 사용하는.

새로운 실험을 구현하려는 application 은 django 기반이었기에, 아래와 같은 script 를 작성했다.

```python
import time

import django
django.setup()

from google.cloud import bigquery
from .models import Model

BATCH_SIZE = 1000


client = bigquery.Client(project="...")
query_job = client.query("...")

batch = []
batch_num = 0

def sink():
    global batch_num
    batch_num += 1
    
    # 여러 record 를 한번의 insert 로 처리한다.
    # insert into ... values ... 에 여러 tuple 을 적는 것.
    # 
    # ignore_conflicts=True 이므로, duplicate-key error 등이 발생해도, 무시한다.
    Model.objects.bulk_create(batch, ignore_conflicts=True)
    
    print(f"Batch {batch_num}: {len(batch)} records")

# jobs.getQueryResults 에서 pagination 을 지원한다.
# pageToken, maxResults 를 parameter 로 받는다.
# https://docs.cloud.google.com/bigquery/docs/reference/rest/v2/jobs/getQueryResults
# 
# .result() 는 RowIterator 를 return 하는데,
# 개가 내부적으로 그 jobs.getQueryResults 를 써서 pagination 해준다.
# 그러니 query result 가 크더라도, memory 에는 page 만큼만 올라간다.  
for row in query_job.result():
    batch.append(Model(**row))
    if len(batch) >= BATCH_SIZE:
        sink()
        batch = []
        time.sleep(1)
if batch:
    sink()

print(f"Done! Total {batch_num} batches.")
```

우려되는 부분들이 있는 rough 한 script 이나, 당장에 필요한 작업을 마치는 데는 문제가 없었다.

# 생각

mysql table 에 primary key (혹은 unique key) 가 있음을 가정했다. 이 글에서 다룬 작업이, 크게 봤을 때 feature store 에 해당하는 얘기라 생각 하는데, 그 쪽에서 그렇게 어색한 가정은 아닐 것이다. entity 개념이 명확하고, entity 에 대한 feature 를 얘기하는 것이니, entity 의 id 가 있을 것이고, 그게 pk 가 될 것이다.

지금 구현에서는 bigquery 에서 record 가 delete 되어도, 그게 mysql 에 반영되지 못하고 계속 남아있게 된다. bigquery 내에서 직전 result 와 이번 result 의 diff 를 따져서 added/updated/deleted 로 분류하여 처리해야 할 것이다.

한번의 실행이 여러 batch 를 처리 하는데, 중간에 어떠한 이유로 실패 하고 재개 했을 때, 이미 처리한 batch 는 넘어가고 / 남은 batch 들만 처리하면 좋을 것이다.
