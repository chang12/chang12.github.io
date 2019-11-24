---
layout: post
title: "Cloud Composer 에서 Airflow Web Server REST API 로 외부에서 DAG 트리거하기"
tags: [Cloud Composer, Airflow]
---

이전까지는 생성자에 `schedule_interval` 을 적어서 DAG 를 주기적으로 실행했습니다. 그러던 중 Airflow 외부의 배치 프로그램이 끝날 때 마다 DAG 를 Trigger 하고 싶은 상황이 생겼습니다. 방법을 찾아보다가 [Triggering DAGs (workflows)](https://cloud.google.com/composer/docs/how-to/using/triggering-with-gcf) 문서를 발견했고 적용해봤습니다. 

문서에서는 Cloud Function 으로 Trigger 하는 방법을 소개하지만, 일반적인 Airflow 외부 환경에서 사용할 수 있는 방법입니다. Airflow Web Server 에서 제공하는 [REST API](https://airflow.apache.org/api.html) 를 호출해서 DAG Run 을 Trigger 합니다. REST API 는 문서와 URL 에 적혀있듯이, 아직 Experimental 단계라서 추후 바뀔 수 있겠습니다.

## 테스트 DAG

[문서의 trigger\_response\_dag.py](https://cloud.google.com/composer/docs/how-to/using/triggering-with-gcf#trigger_response_dagpy) 코드를 참고해서 테스트때 쓸 DAG 를 작성합니다.

```python
from datetime import datetime

from airflow import models
from airflow.operators.bash_operator import BashOperator

with models.DAG('dag_trigger_test',
                schedule_interval=None,
                start_date=datetime(2019, 10, 31),  # DAG 의 start_date 는 필수이니 적당히 아무 값으로 채워봅니다.
                catchup=False) as dag:
    a = BashOperator(task_id='test',
                     # DAG 실행에 필요한 설정값을 REST API 호출 때 HTTP Body 에 적어주면,
                     # Airflow 의 Jinja Templating 의 Variable 을 통해 획득할 수 있습니다.
                     {% raw %}bash_command='echo date_kr : {{ dag_run.conf.get("date_kr") }}'){% endraw %}
```

## Cloud IAP 와 Client ID

[Overview of Cloud Composer 문서의 아키텍처 그림](https://cloud.google.com/composer/docs/concepts/overview#architecture) 을 보면, Airflow Web Server 에 접속하기 위한 인증은 Cloud IAP (Identity-Aware Proxy) 가 관리합니다.

<p align="center"><img src="https://cloud.google.com/composer/docs/images/architecture.svg"/></p>

[Cloud IAP 문서](https://cloud.google.com/iap/docs/authentication-howto#authenticating_from_a_service_account) 를 보니, Cloud IAP 가 관리하는 resource 에 접속하기 위해 인증할 때 resource 의 Client ID 가 필요합니다. 우리가 소유한 Project 의 resource 라면 Cloud IAP Console 에서 Client ID 를 획득할 수 있습니다. 하지만 아키텍처 그림에서 보이듯이, Cloud Composer 로 생성한 Airflow Web Server 는 GCP 에서 관리하는 Tenant Project 의 App Engine 에 배포됩니다. 

그래서 [Getting the client ID](https://cloud.google.com/composer/docs/how-to/using/triggering-with-gcf#getting_the_client_id) 문서에서도, 인증 없이 resource 접속 -> 리다이렉트 -> 리다이렉트된 URL 에서 Client ID 를 알아내는 우회적인 방법을 소개합니다. 문서에서는 Python 코드를 소개하지만, 코드 없이 브라우저에서도 값을 알아낼 수 있습니다. Gmail 로그아웃 후 브라우저에서 Airflow Web Server 에 접속하면 -> Gmail 로그인 페이지로 리다이렉트 되는데 -> 이때 리다이렉트 된 URL 의 `client_id` query parameter = Client ID 입니다.

## Service Account

외부 클라이언트는 Service Account 의 JSON Key 파일을 사용해서 인증합니다. Composer 가 관리하는 resource 에 접속할 수 있는 Role 이 필요할텐데, [Overview of Cloud Composer](https://cloud.google.com/composer/docs/concepts/overview#app-engine) 문서에서 `Composer User` Role 을 주면 된다고 소개합니다.

> To grant access only to the Airflow web server, you can assign the composer.user role, or you can assign different Cloud Composer roles that provide access to other resources in your environment.

만약 Role 을 부여하지 않으면 에러가 발생합니다. 

> Exception: Service account xxx@yyy does not have permission to access the IAP-protected application.

## Authentication

[python-docs-samples repository 의 composer\_storage\_trigger.py](https://github.com/GoogleCloudPlatform/python-docs-samples/blob/cf121715f3bd6d1f0916b966bfcaf0dcf769cd1a/functions/composer/composer_storage_trigger.py) 예시 코드를 참고하여 좀 더 간단하게 정리해봤습니다. [Preparing to make an authorized API call](https://developers.google.com/identity/protocols/OAuth2ServiceAccount#authorizingrequests) 문서의 예시 코드 중 HTTP/REST 단락을 보면 관련하여 좀 더 상세한 설명을 확인할 수 있습니다.

- Service Account Credetials JSON 파일로 Bootstrap Credentials 생성
- `token_uri`, `additional_claims` 값을 바꾼 새로운 Credentials 생성
- 새로운 Credentials 로 sign 한 JWT 생성
- 생성한 JWT 로 Google OAuth 2.0 Authorization Server 에 요청
- 응답에서 OpenID Connect token 획득

```
requests_toolbelt==0.9.1
google-auth==1.6.2
```

```python
from google.auth.transport.requests import Request
from google.oauth2._client import _JWT_GRANT_TYPE, _token_endpoint_request
from google.oauth2.service_account import Credentials

client_id = '<client_id>'
oauth_token_uri = 'https://www.googleapis.com/oauth2/v4/token'
path_to_json = '<path_to_json>'

# service account credentials 파일로 bootstrap credentials 을 생성합니다.
bootstrap_credentials = Credentials.from_service_account_file(path_to_json)
signer_email = bootstrap_credentials.service_account_email
signer = bootstrap_credentials.signer

# OAuth 2.0 service account credentials 을 생성합니다.
# token_uri 값을 바꾸고, additional_claims 을 추가합니다.
service_account_credentials = Credentials(
    signer, signer_email, oauth_token_uri,
    additional_claims={'target_audience': client_id}
)

# OpenID Connect token 을 획득합니다.
service_account_jwt = service_account_credentials._make_authorization_grant_assertion()
body = {'assertion': service_account_jwt, 'grant_type': _JWT_GRANT_TYPE}
token_response = _token_endpoint_request(Request(), oauth_token_uri, body)
google_open_id_connect_token = token_response['id_token']
```

## Airflow Web Server REST API 호출

바로 앞에서 획득한 OpenID Connect token 을 HTTP Header 에 Bearer token 으로 적어주면, Cloud IAP 로 보호되는 Airflow Web Server 의 REST API 를 호출할 수 있습니다. 앞에서와 마찬가지로 [composer\_storage\_trigger.py](https://github.com/GoogleCloudPlatform/python-docs-samples/blob/cf121715f3bd6d1f0916b966bfcaf0dcf769cd1a/functions/composer/composer_storage_trigger.py) 예시 코드를 참고했습니다.

```python
import requests

web_server_id = '<web_server_id>'  # Airflow Web Server = https://<web_server_id>.appspot.com
dag_name = 'dag_trigger_test'
url = f'https://{web_server_id}.appspot.com/api/experimental/dags/{dag_name}/dag_runs'
# 획득한 OpenID Connect token 을 Bearer token 으로.
headers = {'Authorization': f'Bearer {google_open_id_connect_token}'
data = {'conf': {'date_kr': '2019-11-24'}}

resp = requests.request('POST', url, headers=headers, json=data)

if resp.status_code == 403:
    raise Exception(f'Service account {signer_email} does not have permission to '
                    f'access the IAP-protected application.')
elif resp.status_code != 200:
    raise Exception(f'Bad response from application: {resp.status_code} / {resp.headers} / {resp.text}')
```

실행 후 Airflow Web Server 에 접속해서 Task Log 를 확인해보니, DAG 가 정상적으로 실행되었습니다.

```
[2019-11-24 09:10:50,581] {base_task_runner.py:101} INFO - Job 45968: Subtask test [2019-11-24 09:10:50,578] {bash_operator.py:101} INFO - Temporary script location: /tmp/airflowtmp27x0r093/testb_o3ycl3
[2019-11-24 09:10:50,584] {base_task_runner.py:101} INFO - Job 45968: Subtask test [2019-11-24 09:10:50,583] {bash_operator.py:111} INFO - Running command: echo date_kr : 2019-11-24
[2019-11-24 09:10:51,251] {base_task_runner.py:101} INFO - Job 45968: Subtask test [2019-11-24 09:10:51,251] {bash_operator.py:120} INFO - Output:
[2019-11-24 09:10:51,260] {base_task_runner.py:101} INFO - Job 45968: Subtask test [2019-11-24 09:10:51,260] {bash_operator.py:124} INFO - date_kr : 2019-11-24
[2019-11-24 09:10:51,274] {base_task_runner.py:101} INFO - Job 45968: Subtask test [2019-11-24 09:10:51,264] {bash_operator.py:128} INFO - Command exited with return code 0
```

## 레퍼런스

- [Cloud Composer Triggering DAGs (workflows)](https://cloud.google.com/composer/docs/how-to/using/triggering-with-gcf)
- [Airflow REST API Reference](https://airflow.apache.org/api.html)
- [Overview of Cloud Composer : Architecture](https://cloud.google.com/composer/docs/concepts/overview#architecture)
- [Overview of Cloud Composer : App Engine](https://cloud.google.com/composer/docs/concepts/overview#app-engine)
- [Cloud IAP Programmatic authentication](https://cloud.google.com/iap/docs/authentication-howto#authenticating_from_a_service_account)
- [python-docs-samples repository 의 composer\_storage\_trigger.py](https://github.com/GoogleCloudPlatform/python-docs-samples/blob/cf121715f3bd6d1f0916b966bfcaf0dcf769cd1a/functions/composer/composer_storage_trigger.py)
- [Preparing to make an authorized API call](https://developers.google.com/identity/protocols/OAuth2ServiceAccount#authorizingrequests)
