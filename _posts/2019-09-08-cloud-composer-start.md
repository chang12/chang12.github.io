---
layout: post
title: Cloud Composer 시작하기
tags: [Cloud Composer, Apache Airflow]
---

회사 업무때 OLAP 로 BigQuery 를 쓰고 있습니다. 그래서 ETL 작업을 크게 **BigQuery 로 옮기는 작업**과, **BigQuery 내에서의 작업**으로 두 종류로 나눌 수 있습니다. 현재는 두 종류 작업 모두 Apache Spark 에 의존하는 Scala 코드로 작성되어 있고, Amazon EMR 클러스터에 `spark-submit` 로 제출되어 실행됩니다. 그런데 BigQuery 내에서의 작업은 컴퓨팅 자원이 적게 들기 때문에 굳이 Apache Spark 에 의존할 필요가 없고, 데이터 분석가 분들도 작업을 기술할 수 있도록 Python 코드로 작성되면 좋을 것 같습니다. 그래서 Apache Airflow 를 쓰면 좋겠다는 생각이 들었고, Google Cloud 가 제공하는 Managed Airflow 서비스인 [Cloud Composer](https://cloud.google.com/composer/) 를 써보려고 합니다.

BigQuery 내에서의 작업을 크게 두 종류로 나누고, 그 중 더 간단한 첫번째 종류의 작업을 Cloud Composer 를 써서 처리해봅니다.

- **Unpartitioned 테이블을 Overwrite 하는 작업.**
- Date Partitioned 테이블의 Partition 을 Overwrite 하는 작업.

## Environment 생성

Cloud Composer 클러스터(?) 를 **Environment** 라고 부릅니다. [Composer Console 에서 CREATE 버튼 클릭 후, 설정값을 넣고, Environment 를 생성하면](https://console.cloud.google.com/composer/environments), Google Kubernetes Engine 으로 Kubernetes Cluster 를 띄우고, Airflow 의 각종 구성 요소들을 알아서 생성해줘서 편리합니다.

몇가지 설정값들은 지정하고,

- `Name` : 적당한 이름으로.
- `Node count` : 3 (기본값)
- `Location` : us-central1
- `Image version` : composer-1.7.5-airflow-1.10.2 (생성 시각 기준 최신)
- `Python version` : 3

나머지는 기본값을 그대로 사용했습니다.

- `Zone` : us-central1-b (Location 내에서 랜덤 지정인 듯)
- `Machine type` : n1-standard-1 (1 vCPU, 3.75 GB 메모리)
- `Disk size` : 100 GB
- `OAuth Scopes` : auth/cloud-platform (뭔지 모르겠음)
- `Service account` : {...}-compute@developer.gserviceaccount.com (뭔지 모르겠음)
- `Tags` : None
- `Additional Features` : 네트워크 설정 등등

생성 후 Running 상태로 전이할 때 까지 ~ 15분 정도 걸립니다.

![images/2019-09-08-pic1-environment.png](https://raw.githubusercontent.com/chang12/chang12.github.io/master/images/2019-09-08-pic1-environment.png)

- `Airflow webserver` : 웹인데 다양한 기능들이 있습니다.
- `Logs` : Stackdriver Logging 로 넘어갑니다.
- `DAGs folder` : Cloud Storage 로 넘어갑니다. 뒤에서 DAG 를 추가할 때 접속합니다.

## 개발 환경

IDE 로 PyCharm 을 사용합니다. 로컬 개발 머신에서 실행하진 않을 것이지만, Dependency 가 충족되어 있어야 코드 편집하기 편리하므로, virtualenv 를 생성하고 airflow package 를 설치합니다.

```
SLUGIFY_USES_TEXT_UNIDECODE=yes pip install apache-airflow[gcp]==1.10.2
```

- 앞서 Environment 를 만들 때 Image version 이 composer-1.7.5-airflow-1.10.2 였습니다. 그래서 airflow package 의 버전도 1.10.2 로 맞춰줍니다.
- [Extra Packages 문서](https://airflow.apache.org/installation.html) 를 보니 subpackage 중 gcp 가 있습니다. 정확히 뭘 해줄지 모르겠지만, `apache-airflow[gcp]` 로 명시합니다.
- `SLUGIFY_USES_TEXT_UNIDECODE=yes` 로 변수 값을 명시해주지 않으면, GPL 관련 에러가 발생하는데, 무슨 뜻인지 모르겠습니다. stack overflow 답변에서 시키는대로 해서 넘어갑니다.

## DAG 작성

[Writing DAGs (workflows)](https://cloud.google.com/composer/docs/how-to/using/writing-dags) 문서를 읽고 따라서 DAG 를 작성했습니다. [Structuring a DAG](https://cloud.google.com/composer/docs/how-to/using/writing-dags#structure) 를 따라 DAG context manager 를 열고, [Google Cloud Platform Operators](https://cloud.google.com/composer/docs/how-to/using/writing-dags#gcp_operators) 에서 소개한 `BigQueryOperator` 를 사용해서 쿼리 결과를 별도의 테이블로 Overwrite 합니다.

```python
import datetime

from airflow import models
from airflow.contrib.operators.bigquery_operator import BigQueryOperator

SQL = """
select ...
"""

# start_date 는 필수입니다. "없으면 Broken DAG: [...] Task is missing the start_date parameter" 에러 발생.
# catchup=False 로 주지 않고, 기본값인 True 로 넣으면, start_date ~ DAG 추가 시각까지 schedule_interval 개수 만큼 과거를 다 실행해버립니다.
# 이번에 추가할 Task 는 Unpartitioned 테이블을 Overwrite 하는 작업이라, DAG 추가 시각부터만 잘 실행되는 걸로 충분합니다.
with models.DAG('<dag_id>',
                schedule_interval=datetime.timedelta(hours=1),
                start_date=datetime.datetime(2019, 9, 7, 0, 0, 0),
                catchup=False) as dag:
    # context manager 안에서 task 를 생성하면, 자동으로 DAG object 에 추가됩니다.
    # task_id 는 필수입니다. 없으면 "Broken DAG: [...] Argument ['task_id'] is required" 에러 발생.
    task = BigQueryOperator(task_id='<task_id>',
                            sql=SQL,
                            destination_dataset_table='<dataset>.<table>',
                            write_disposition='WRITE_TRUNCATE',
                            use_legacy_sql=False)

```

## DAG 업로드

![images/2019-09-08-pic1-environment.png](https://raw.githubusercontent.com/chang12/chang12.github.io/master/images/2019-09-08-pic1-environment.png)

DAGs folder 를 클릭하면, Environment 생성할 때 자동으로 생성된 Bucket 경로의 Cloud Storage Console 로 이동합니다. [Quickstart](https://cloud.google.com/composer/docs/quickstart#uploading_the_dag_to) 문서를 참고해서 위에서 작성한 DAG .py 파일을 업로드 합니다. 업로드 후 1-2분 안에 반영되어 Airflow webserver 에서 확인할 수 있습니다.

> After you upload your DAG, Cloud Composer adds the DAG to Airflow and schedules the DAG immediately. It might take a few minutes for the DAG to show up in the Airflow web interface.

수 시간이 지난 뒤, Airflow webserver 의 Browse 탭의 Task Instances 메뉴로 접속하면, Task 선언때 적은대로 1시간 마다 주기적으로 작업이 실행되는 것을 확인할 수 있습니다.

![images/2019-09-08-pic2-task-instances.png](https://raw.githubusercontent.com/chang12/chang12.github.io/master/images/2019-09-08-pic2-task-instances.png)

## 레퍼런스

- [Airflow 문서의 Extra Packages](https://airflow.apache.org/installation.html)
- [Error while install airflow: By default one of Airflow's dependencies installs a GPL](https://stackoverflow.com/questions/52203441/error-while-install-airflow-by-default-one-of-airflows-dependencies-installs-a)
- [Writing DAGs (workflows)](https://cloud.google.com/composer/docs/how-to/using/writing-dags)
- [Quickstart 중 Uploading the DAG to Cloud Storage](https://cloud.google.com/composer/docs/quickstart#uploading_the_dag_to)

***

#### Cloud Composer 관련 다른 포스트

- Cloud Composer 시작하기
- [Cloud Composer 를 업무에 처음 도입하며 정리한 몇가지 생각들](https://chang12.github.io/cloud-composer-some-thought/)
- [Node Pool 로 KubernetesPodOperator 를 위한 독립된 실행 환경 구성하기](https://chang12.github.io/cloud-composer-node-pool-kubernetes-pod-operator/)
- [Cloud Composer 에서 Airflow Web Server REST API 로 외부에서 DAG 트리거하기](https://chang12.github.io/composer-trigger-dag/)
