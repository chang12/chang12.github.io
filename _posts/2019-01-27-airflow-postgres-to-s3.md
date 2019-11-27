---
layout: post
title: Airflow 에 PostgresToS3Operator 만들기
tags: [Apache Airflow, AWS]
---

ETL 작업을 어떻게 관리하는게 좋을지 고민입니다. 검색했을 때나 주변 분들 얘기를 들어봤을 때 [Apache Airflow](https://airflow.apache.org/) 가 많이 쓰이는 것 같습니다. 매니지드 서비스로는 GCP 에서는 [Cloud Composer](https://cloud.google.com/composer/), AWS 에서는 [AWS Data Pipeline](https://aws.amazon.com/ko/datapipeline/) 가 있습니다. 업무 인프라의 대부분이 AWS 라서 Data Pipeline 에 눈이 갔는데 아쉽게도 서울 리전에는 아직 제공되지 않습니다.

그래서 직접 EC2 를 띄워서 Airflow 를 사용해보는 연습을 하기로 결정했고, 간단한 Use Case 로 PostgreSQL 테이블의 데이터를 주기적으로 select 해서 S3 에 파일로 저장하는 작업을 떠올렸습니다. 

[Tutorial](https://airflow.apache.org/tutorial.html) 문서를 읽어보니 Airflow 로 작업을 정의할 때 사전에 준비된 **Operator** (e.g. BashOperator) 를 활용할 수 있습니다. RDBMS (PostgreSQL) -> S3 export 도 충분히 일반적인 작업이라 생각해서 Operator 가 준비되어 있을 것 같았습니다. 찾아보니 GCP Cloud Storage 로 export 하는 [mysql\_to\_gcs.py](https://github.com/apache/airflow/blob/ecc77c2bf706a61d1dd77b1a95a00b74f29f74e3/airflow/contrib/operators/mysql_to_gcs.py) 와 [postgres\_to\_gcs\_operator.py](https://github.com/apache/airflow/blob/1e79dae06efa43eb26ddbd7ad0de4cc2df65aa57/airflow/contrib/operators/postgres_to_gcs_operator.py) 는 있었습니다. 하지만 아쉽게도 S3 로 export 하는 Operator 는 없었습니다. operators 를 훑어보니 전반적으로 GCP 관련된 게 많았습니다. 

그래서 공부할 겸 [postgres\_to\_gcs\_operator.py](https://github.com/apache/airflow/blob/1e79dae06efa43eb26ddbd7ad0de4cc2df65aa57/airflow/contrib/operators/postgres_to_gcs_operator.py) 를 참고해서 PostgreSQL -> S3 로 export 하는 Operator 를 작성해봤고, 이번 글에서는 그 과정 중에 몇 가지 얘기들을 적어봤습니다.

## EC2 에 Airflow 세팅

Instance Type 은 **r5.xlarge** 를 사용했습니다. 기존의 Zeppelin 을 r4.xlarge 로 사용하고 있었고, 그새 r5 타입이 나왔길래 별 생각없이 같은 xlarge 로 띄워봤습니다. 돌이켜보니 실습 때는 가벼운 SQL 을 실행하기 때문에 과했던 것 같습니다. AMI 도 기존에는 16.04 를 쓰고 있었는데 실험삼아 Ubuntu 18.04 (ami-06e7b9c5e0c4dd014) 로 해봤습니다. 결과적으로 차이는 없었습니다.

타겟 PostgreSQL 은 RDS 로 띄웠습니다. Airflow 가 띄워진 EC2 머신에서 RDS 에 접속할 수 있도록 Airflow EC2 머신의 Security Group 에 대한 inbound rule 을 RDS 의 Security Group 에 추가합니다. 또한 S3 에 파일을 업로드할 것이므로 Airflow EC2 머신의 IAM role 에 S3 에 접근할 수 있는 권한을 추가해줍니다. 실습때는 **AmazonS3FullAccess** policy 를 품은 IAM role 을 만들어 부여했습니다.

[Quick Start](https://airflow.apache.org/start.html) 문서를 참고해서 나머지 Airflow 세팅을 진행합니다.

```bash
export AIRFLOW_HOME=~/airflow

# 아래 export 없이 pip install apache-airflow 하면 예외 발생.
export SLUGIFY_USES_TEXT_UNIDECODE=yes
pip install apache-airflow

# 기본 동작으로 sqlite 파일 생성.
airflow initdb

# airflow webserver -p 8080
# nohup 으로 background 에서 서버 실행.
nohup airflow webserver -p 8080 &

# (Optional) postgresql-client 로 CLI 에서 RDS PostgreSQL 접속 가능한지 확인.
# psql 명령어 실행 후 password 입력 -> 접속 성공.
sudo apt-get update
sudo apt-get install postgresql-client
psql -h my_rds_endpoint my_database_name my_rds_user
```

## PostgresToS3Operator 작성

[Tutorial](https://airflow.apache.org/tutorial.html) 문서를 읽어보니 Operator 를 써서 Task 를 정의하고, Task 여러 개를 DAG 로 묶는 구조입니다. 문서의 코드를 본따서 DAG 와 Task 를 정의하는 `postgres_to_s3.py` 코드를 `AIRFLOW_HOME/dags` 아래 추가합니다. `default_args` 는 아직 다 이해를 못했으나, Tutorial 코드의 기본 구조를 따르기 위해 그대로 복붙했습니다. 핵심이 되는 **PostgresToS3Operator** 는 이후에 [postgres\_to\_gcs\_operator.py](https://github.com/apache/airflow/blob/1e79dae06efa43eb26ddbd7ad0de4cc2df65aa57/airflow/contrib/operators/postgres_to_gcs_operator.py) 의 PostgresToGoogleCloudStorageOperator 를 참고해서 작성합니다.

```python
default_args = {
    'owner': 'airflow',
    'depends_on_past': False,
    'start_date': datetime.datetime(2015, 6, 1),
    'email': ['airflow@example.com'],
    'email_on_failure': False,
    'email_on_retry': False,
    'retries': 1,
    'retry_delay': datetime.timedelta(minutes=5),
    # 'queue': 'bash_queue',
    # 'pool': 'backfill',
    # 'priority_weight': 10,
    # 'end_date': datetime(2016, 1, 1),
}

dag = DAG('exercise_postgres_to_s3_dag', default_args=default_args)

task_id = 'postgres_to_s3'
sql = 'select * from ... limit 20'
bucket = 'my-bucket-name'
filename = 'my-file-name'
t1 = PostgresToS3Operator(task_id='postgres_to_s3', dag=dag, sql=sql, bucket=bucket, filename=filename)
```

### PostgreSQL connection 설정

[Managing Connections](https://airflow.readthedocs.io/en/stable/howto/manage-connections.html) 문서에서 환경변수로 connection 을 설정하는 방법을 소개합니다. Operator, Hook 코드에서 `_conn_id` suffix 가 붙은 이름의 파라미터에 argument 로 `abc_def` 를 넘기면 `AIRFLOW_CONN_` postfix 를 붙인 `AIRFLOW_CONN_XYZ_DEF` 이름의 환경변수 값을 참조하는 방식입니다. PostgreSQL DBMS 에 접속할 때도 마찬가지 방식을 사용하고, `postgres_conn_id` 파라미터의 기본값으로 **'postgres_default'** 을 사용하므로 이에 맞춰 ~/.bashrc 에 환경변수를 선언합니다.

```bash
export AIRFLOW_CONN_POSTGRES_DEFAULT='postgresql://<user>:<password>@<rds_endpoint>:<port>/<database>'
```

또한 PostgreSQL 에 대한 DB API 2.0 구현체인 **psycopg2** 패키지를 설치합니다.

```
pip install psycopg2
```

### S3 에 파일 업로드

PostgresToGoogleCloudStorageOperator 는 GCS 에 파일 업로드하는 코드를 `_upload_to_gcs` 라는 이름의 메서드로 분리해서 작성했습니다. 이를 본따서 `_upload_to_s3` 메서드를 작성합니다. 이때 Airflow 에서 제공하는 [S3Hook](https://github.com/apache/airflow/blob/52ee34c5fc7818797e0a7463d896e80e1482cbfd/airflow/hooks/S3_hook.py) 을 활용합니다. 동작의 멱등성을 위해 `replace` parameter 의 값을 True 로 넘깁니다.

```python
from airflow.hooks.S3_hook import S3Hook

def _upload_to_s3(self, files_to_upload):
    """
    Upload all of the file splits to S3.
    """
    hook = S3Hook()
    for k, tmp_file_handle in files_to_upload.items():
        hook.load_file(filename=tmp_file_handle.name, key=k, bucket_name=self.bucket, replace=True)
```

실행하기 위해선 AWS 의 python sdk 인 **boto3** 패키지 설치가 선행되어야 합니다.

```
pip install boto3
```

### `convert_types` 메서드 간소화

PostgresToGoogleCloudStorageOperator 코드를 읽어보면 추후 Cloud Storage -> BigQuery 로 import 할 것을 염두에 두고 작성된 것을 느낄 수 있습니다. 그래서 추후 import 할 때 PostgreSQL 와 BigQuery 간의 data types 차이로 인한 문제가 발생하지 않도록, 파일로 적기 직전에 BigQuery 에 적절한 data types 으로 변환하는 `convert_types` 메서드를 호출합니다. 이때 timestamp 타입을 epoch second 로 변환하는 데 이게 마음에 들지 않으므로 정해진 포맷의 문자열로 변환하고, 나머지 변환은 모두 배제합니다.

```python
@classmethod
def convert_types(cls, value):
    if type(value) is datetime.datetime:
        return value.strftime('%Y-%m-%d %H:%M:%S')
    return value
```

### 한글 직렬화 문제 없게 `_write_local_data_files` 수정

실무에서는 PostgreSQL 에 한글 데이터가 많이 포함되어 있습니다. 따라서 utf-8 인코딩된 문자열이 문제없이 직렬화 되도록 `_write_local_data_files` 메서드의 코드를 수정합니다.

```python
# comment 가 수정 전.

# handle = NamedTemporaryFile(delete=True)
handle = NamedTemporaryFile(mode='w', encoding='utf-8', delete=True)

# s = json.dumps(row_dict, sort_keys=True)
# if PY3:
#     s = s.encode('utf-8')    
# tmp_file_handle.write(s)
# tmp_file_handle.write(b'\n')
tmp_file_handle.write(json.dumps(row_dict, ensure_ascii=False, sort_keys=True))
tmp_file_handle.write('\n')
```

### `execute` 메서드 간소화

Operator 의 main 함수에 해당하는 `execute` 메서드에서도 Cloud Storage / BigQuery 에 특화된 코드들을 제거하고 간단히 합니다.

```python
def execute(self, context):
    cursor = self._query_postgres()
    files_to_upload = self._write_local_data_files(cursor)
    self._upload_to_s3(files_to_upload)
```

## 테스트

Airflow 에서 제공하는 **test** 명령어로 테스트합니다. `default_args` 에 대한 미진한 이해로... 일단 임의의 날짜를 나타내는 문자열을 argument 로 넘겨서 양식을 맞춰줍니다. SQL 에 의도한대로 20 rows 를 읽어서 파일 1개로 저장하고 S3 에 파일이 잘 업로드 되었음을 확인할 수 있습니다.

```shell
> airflow test exercise_postgres_to_s3_dag postgres_to_s3 2015-06-01

(각종 로그 출력...)
[2019-01-27 16:40:16,978] {postgres_to_s3.py:167} INFO - Received 20 rows over 1 files
```

## 마치며

AWS EC2 머신에 Airflow 를 간단히 설치해보고, 기존 Operator 를 참고해서 원하는 Operator 를 작성해보는 경험을 해볼 수 있었습니다.

하지만 아직 갈 길이 멀게 느껴집니다. 당장에 Airflow 를 익히는데도 많은 시간이 필요해보입니다. 단적인 예로 우선 원하는 기능을 구현하는 데 초점을 맞췄으나, 글을 작성하며 찾아보니 PostgreSQL 에 쿼리 + 로컬 파일 저장을 묶은 [postgres_hook.py](https://github.com/apache/airflow/blob/c27098b8d31fee7177f37108a6c2fb7c7ad37170/airflow/hooks/postgres_hook.py) 가 있는 것을 뒤늦게 발견했습니다. 또한 Workflow Management 계열에 Luigi 라는 프레임워크도 있다고 하니 어느게 좋을지 궁금해지기도 합니다. 그리고 Airflow 와 Spark 를 어떻게 결합할 까 고민하다보면 둘 다 Kubernetes 환경에서의 실행을 지원하다보니... EMR 로 넘어가려던 계획을 EKS + EC2 로 재조정할까 싶기도 합니다. 여러모로 이번 글은 많은 여운을 남겼습니다.

## 레퍼런스

* [Airflow Docs: Quick Start](https://airflow.apache.org/start.html)
* [Airflow Docs: Tutorial](https://airflow.apache.org/tutorial.html)
* [Airflow Docs: Managing Connections](https://airflow.readthedocs.io/en/stable/howto/manage-connections.html)
* [Airflow GitHub: postgres\_to\_gcs\_operator.py](https://github.com/apache/airflow/blob/1e79dae06efa43eb26ddbd7ad0de4cc2df65aa57/airflow/contrib/operators/postgres_to_gcs_operator.py)
* [Airflow GitHub: S3_hook.py](https://github.com/apache/airflow/blob/52ee34c5fc7818797e0a7463d896e80e1482cbfd/airflow/hooks/S3_hook.py)
