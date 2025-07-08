---
layout: post
title: "Cloud Composer 를 업무에 처음 도입하며 정리한 몇가지 생각들"
tags: [Cloud Composer, Apache Airflow]
---

Cloud Composer 를 업무에 적용해보기 위해 처음 학습한 내용을 [Cloud Composer 시작하기](https://chang12.github.io/cloud-composer-start/) 글에서 정리했었습니다. 학습한 내용들을 가지고 이후 실제로 업무에 (= 데이터 파이프라인 개발) 적용했고, 그 과정중에 몇가지 고민하고 정리한 내용들이 있었습니다.

## SQL 과 Python 스크립트는 별개의 파일로 분리

우선 BigQuery 내의 데이터 이동만 DAG 로 기술해서 Cloud Composer 로 실행하고 있습니다. 그래서 [BigQueryOperator](https://airflow.apache.org/_api/airflow/contrib/operators/bigquery_operator/index.html#airflow.contrib.operators.bigquery_operator.BigQueryOperator) 에 `sql` argument 를 넘겨 SQL 을 실행하고, 결과를 BigQuery 테이블로 저장합니다. 이때 작업에 따라 `sql` argument 문자열이 꽤 길어질 수 있습니다. 그렇게 길어진 SQL 을, DAG 를 기술하는 Python 스크립트에 함께 적으면, 번거로워집니다.

- BigQuery Console 에서 SQL 을 작성->실행->코드로 옮길때나, 반대로 코드의 SQL 을 Console 로 복사->실행할 때, Python 스크립트에서 SQL 부분만 긁어야 합니다.
- Python 코드를 편집할때는 1 tab = 4 spaces 로 쓰고 있고, SQL 은 1 tab = 2 spaces 로 쓰고 있습니다. 따라서 2가지가 하나의 파일에 혼재할 경우, PyCharm 같은 IDE 를 써서 편집할 때, SQL 의 1 tab 도 4 spaces 가 되어버립니다.

그래서 SQL 은 별도의 `.sql` 파일로 관리하고, DAG 에서 파일을 읽고, 동적으로 채울 값이 있다면 원하는 값으로 채워서 사용합니다. 이를 위해 간단한 Python 함수를 작성했습니다.

```python
def read_sql(file_name):
    with open(file_name, 'r') as file:
        s = file.read()
    return s
```

SQL 은 아래처럼 별도의 파일로 작성하고, 동적으로 채울 값은 `{변수명}` 으로 자리를 만들어놓고, 필요할 때 값을 넣어 사용합니다.

```sql
-- 예를 들어 파일 상대 경로가 x.sql 라면,
-- read_sql('x.sql').format(date_kr='2019-09-11') 로 SQL 을 완성합니다.

select
  count(1) as c
from
  tada.ride
where
  date_kr = '{date_kr}'
```

## 파일 경로 정할 때 `DAGS_FOLDER` 환경 변수 사용

Cloud Composer 는 Cloud Storage 에 올라간 DAG 스크립트를 읽어갑니다. DAG 스크립트를 포함한 전체 디렉토리 구조는 아래처럼 구성해서 사용하고 있고, 이는 Cloud Storage 에서도 동일합니다.

```
dags/
    lib/
        __init__.py
        utils.py
        ...
    sql/
        a.sql
        b.sql
        ...
    dag_1.py
    dag_2.py
    ...
```

이때 DAG 스크립트에서 SQL 파일을 읽을때, 상대경로로 (e.g. `sql/a.sql`) 읽으면 될거라고 생각했습니다. 하지만 에러가 발생합니다. 원인을 알기 위해 몇가지 값을 출력하는 Task 를 작성해서, 임시로 DAG 에 추가하고, 실행합니다.

```python
import os

from airflow.operators.python_operator import PythonOperator


def test_callable():
    print(os.getcwd())
    print(os.path.abspath(__file__))
    print(read_sql('sql/a.sql'))

test = PythonOperator(task_id='test', python_callable=test_callable)
```

Task 는 실패했고, Log 를 확인해보니,

- `os.getcwd()` 값은 `/home/airflow`
- `os.path.abspath(__file__)` 값은 `/home/airflow/gcs/dags/dag_1.py`
- `read_sql('sql/a.sql')` 은 `No such file or directory: 'sql/assigned_area.sql'` 로 실패합니다.

상대 경로로 파일을 읽으려고 시도하면, current working directory 가 기준 경로가 되고, 그 값은 DAG 스크립트의 위치와 다르기 때문에, 실제 SQL 파일의 경로가 아닌 잘못된 경로를 읽으려고 해서 실패하게 됩니다.

이 경우 Airflow 에서 기본 설정으로 제공해주는 `DAGS_FOLDER` 환경 변수를 사용하면 편리합니다. 앞서 소개한 디렉토리 구조로 DAG 스크립트를 넣었을 때, `DAGS_FOLDER` 값은 `/home/airflow/gcs/dags` 가 됩니다. 따라서 그 경로를 기준으로 사용하면, 올바른 SQL 파일 경로를 얻을 수 있습니다.

```python
task_a = BigQueryOperator(
    task_id='a', 
    sql=read_sql(os.path.join(os.environ['DAGS_FOLDER'], 'sql/a.sql')),
    ...
```

## Task 실패하면 Slack 알림

[DAG 문서](https://airflow.readthedocs.io/en/stable/_api/airflow/models/dag/index.html?highlight=dag#airflow.models.dag.DAG) 를 보면, 생성자에서 `on_failure_callback` argument 를 받습니다.

> **on\_failure\_callback** (callable) – A function to be called when a DagRun of this dag fails. A context dictionary is passed as a single parameter to this function.

DAG 실행이 실패했을 때 실행되는 python callable 입니다. 사내 메신저로 Slack 을 쓰고 있어서, DAG 실패 때 Slack 으로 알림을 받으면 좋을 것 같으니, Slack 으로 알림 보내는 python callable 을 작성해서 넘기면 좋겠습니다. 검색해보니 [Integrating Slack Alerts in Airflow](https://medium.com/datareply/integrating-slack-alerts-in-airflow-c9dcd155105) 라는 좋은 글이 있어 참고했습니다. 

- 해당 글에서 `SlackAPIPostOperator` 를 쓰는 방법과 `SlackWebhookOperator` 를 쓰는 방법 2가지를 소개하고, 전자는 Slack 팀에서 추천하는 방식이 아니라고 알려주고 있습니다. 전자는 token 을 만들고 여러 권한을 추가하는 방식인데, 그보다는 정확히 하나의 작업을 (e.g. 특정 channel 에 메시지 POST) 위한 webhook 을 만들어 쓰는 후자의 방법이 나은 듯 합니다. Slack 팀의 생각에 공감합니다. Webhook 만드는 것 관련해서는 [Incoming Webhooks](https://api.slack.com/incoming-webhooks) 문서를 참고합니다. 이후 [Integrating Slack Alerts in Airflow](https://medium.com/datareply/integrating-slack-alerts-in-airflow-c9dcd155105) 의 내용을 참고해서 Slack 메시지 전송을 위한 Airflow Connection 을 생성합니다.
- 해당 글에서는 `SlackWebhookOperator` 를 사용하는데, Task 를 만들게 아니라 특정 channel 에 메시지를 보내는 단위 작업만 실행할 거라서, Hook 레이어의 코드를 실행하는 것으로 충분해보입니다. 이 경우에는 `SlackWebhookHook` 이 됩니다.
- 하지만 `SlackWebhookHook` 은 메시지의 text 만 설정할 수 있어 아쉽습니다. 그래서 `SlackWebhookHook` 의 코드를 참고하여, 직접 HttpHook 을 활용해서, Slack 에 메시지 보내는 python callable 을 작성했습니다. Wehbook 에 보내는 POST 요청의 payload 를 마음대로 수정할 수 있어, [Formatting messages](https://api.slack.com/docs/message-formatting) 문서에서 소개하는 다양한 formatting 들을 사용할 수 있습니다. attachment 를 쓰고 color 값을 명시해서, 메시지 왼쪽 세로줄의 색깔을 달리하는 것을 (성공 = green / 실패 = red), 개인적으로 선호합니다. 

```python
import json

from airflow.hooks.base_hook import BaseHook
from airflow.hooks.http_hook import HttpHook


def task_fail_slack_alert(context):
    # 'slack' 이름으로 Airflow Connection 을 만들었습니다.
    hook = HttpHook(http_conn_id='slack')

    # context 에서 여러 유용한 값들을 제공합니다.
    # 적절히 뽑아서 메시지를 작성합니다.
    return hook.run(
        endpoint=BaseHook.get_connection('slack').password,
        data=json.dumps({
            'text': '<@Uxxxxxxxx>',  # Slack 에서 호출할 유저의 ID.
            'attachments': [{
                'text': '\n'.join((
                    f"*Dag*: {context.get('task_instance').dag_id}",
                    f"*Task*: {context.get('task_instance').task_id}",
                    f"*Execution Time*: {context.get('execution_date')}",
                    f"*Log Url *: {context.get('task_instance').log_url}"
                )),
                'color': '#FF0000'
            }]
        }),
        headers={'Content-type': 'application/json'}
    )

```

실제 Slack channel 에 전송되는 메시지는 아래 이미지처럼 생겼습니다.

![2019-09-22-pic1-slack-alert.png](/images/2019-09-22-pic1-slack-alert.png)

## 레퍼런스

- [Slack API 문서 중 Incoming Webhooks](https://api.slack.com/incoming-webhooks)
- [Integrating Slack Alerts in Airflow](https://medium.com/datareply/integrating-slack-alerts-in-airflow-c9dcd155105)

***

#### Cloud Composer 관련 다른 포스트

- [Cloud Composer 시작하기](https://chang12.github.io/cloud-composer-start/)
- Cloud Composer 를 업무에 처음 도입하며 정리한 몇가지 생각들
- [Node Pool 로 KubernetesPodOperator 를 위한 독립된 실행 환경 구성하기](https://chang12.github.io/cloud-composer-node-pool-kubernetes-pod-operator/)
- [Cloud Composer 에서 Airflow Web Server REST API 로 외부에서 DAG 트리거하기](https://chang12.github.io/composer-trigger-dag/)
