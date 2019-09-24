---
layout: post
title: "SageMaker Get Started 에 더하여 : API Gateway, Lambda 로 배포"
tags: [SageMaker, API Gateway, Lambda]
---

ML 모델을 서비스로 배포하려면 어떻게 하는게 좋을지 궁금해졌습니다. AWS 에는 [Amazon SageMaker](https://aws.amazon.com/ko/sagemaker/) 가 있고, GCP 에는 [AI Platform](https://cloud.google.com/ai-platform/?hl=ko) 이 있습니다. 둘 다 궁금한데 우선 AWS 쪽 부터 파봅니다.

[Developer Guide 의 Get Started](https://docs.aws.amazon.com/sagemaker/latest/dg/gs.html) 를 읽으며 따라갔습니다. 퍼블릭 클라우드에서 AI 의 비중은 앞으로 더 커질 것이고, 따라서 AWS 에서도 신경을 많이 쓰고 있을거라고 생각합니다. 그래서인지 문서가 친절하게 잘 적혀있습니다. 필요한 인프라들을 띄우며 MNIST 데이터셋을 XGBoost 모델로 학습하고, 배포하고, 추론하는데 까지 1시간이 채 안걸렸습니다. [Step 7.1: Validate a Model Deployed to Amazon SageMaker Hosting Services](https://docs.aws.amazon.com/sagemaker/latest/dg/ex1-test-model-endpoint.html) 의 코드를 살짝 수정했습니다.

<p align="center">
  <img src="https://raw.githubusercontent.com/chang12/chang12.github.io/master/images/2019-07-21-pic1-mnist-xgboost-inference.png" alt="2019-07-21-pic1-mnist-xgboost-inference.png"/>
</p>

## ML inference 를 서비스로 만드려면?

[Step 9: Integrating Amazon SageMaker Endpoints into Internet-facing Applications](https://docs.aws.amazon.com/sagemaker/latest/dg/getting-started-client-app.html) 에서 AWS Lambda 를 쓰는 방법을 제안하고 있습니다. 앞에 HTTP endpoint 도 있으면 좋을 것 같습니다. 많이들 사용하는 Amazon API Gateway + AWS Lambda 조합이 생각납니다. 이를 구현하는 여러가지 방법이 가능한데, [chalice](https://github.com/aws/chalice) 프레임워크를 사용해봅니다. Flask 와 유사하게 어플리케이션 코드만 작성하면 chalice 가 필요한 aws 인프라를 구성해줍니다.

<p align="center">
  <img src="https://raw.githubusercontent.com/chang12/chang12.github.io/master/images/2019-07-21-pic2-api-gateway-lambda-sagemaker-endpoint.png" alt="2019-07-21-pic2-api-gateway-lambda-sagemaker-endpoint.png"/>
</p>

## chalice 로 API Gateway + Lambda + SageMaker Endpoint

[chalice README.md 의 Creating Your Project](https://github.com/aws/chalice#creating-your-project) 를 따라 새로운 프로젝트를 위한 디렉토리를 생성합니다.

```
$ chalice new-project your-app-name
```

`requirements.txt` 내용을 수정합니다. SageMaker API 를 호출하기 위한 `boto3`, multipart/form-data 로 올린 jpg 데이터를 뽑아낼 때 쓸 `requests-toolbelt`, 뽑아낸 jpg 데이터를 xgboost 모델의 입력에 맞게 변환할 때 필요한 `Pillow`, `numpy` 를 적습니다.

```
boto3==1.9.191
numpy==1.16.4
Pillow==6.1.0
requests-toolbelt==0.9.1
```

`app.py` 의 내용을 아래 코드로 교체합니다.

```python
import io

import boto3
import numpy as np
from PIL import Image
from chalice import Chalice
from requests_toolbelt import MultipartDecoder

app = Chalice(app_name='your-app-name')
# 'multipart/form-data' 가 꼭 포함되야 합니다. 안 그러면 API Gateway 가 byte 를 manipulate 해서 Pillow 로 처리할 때 에러가 발생합니다.
app.api.binary_types = [
    'application/octet-stream', 'application/x-tar', 'application/zip',
    'audio/basic', 'audio/ogg', 'audio/mp4', 'audio/mpeg', 'audio/wav',
    'audio/webm', 'image/png', 'image/jpg', 'image/jpeg', 'image/gif',
    'video/ogg', 'video/mpeg', 'video/webm',
    'multipart/form-data'
].
sagemaker = boto3.client('sagemaker-runtime')


@app.route('/invoke', methods=['POST'], content_types=['multipart/form-data'])
def invoke():
    decoder = MultipartDecoder(app.current_request.raw_body, app.current_request.headers['content-type'])

    img_bytes = None
    for part in decoder.parts:
        content_disposition = part.headers['Content-Disposition'.encode()].decode()
        # multipart/form-data 에서 img 라는 key 로 jpg 파일을 업로드 하는 상황을 가정합니다.
        # 좀 더 우아하게 처리할 수 있으면 좋을텐데...
        # 참고로 content_disposition 값은 form-data; name="img"; filename="9.jpg" 처럼 적힙니다.
        if "img" in content_disposition:
            img_bytes = part.content
    assert img_bytes is not None

    img = Image.open(io.BytesIO(img_bytes))
    arr = np.array(img) / 255  # gray scale 로 맞춰줍니다.
    csv = ','.join([str(f) for f in arr.flatten().tolist()])  # xgboost 입력에 맞춰 comma separated value 로 만들어줍니다.
    res = sagemaker.invoke_endpoint(EndpointName='your-xgboost-model-endpoint-name',
                                    Body=csv,
                                    ContentType='text/csv',
                                    Accept='Accept')
    label = int(float(res['Body'].read().decode('utf-8')))

    return {'label': label}

```

터미널에서 chalice 명령어로 배포하면, API Gateway 가 생성한 Rest API URL 를 출력해줍니다.

```
$ chalice deploy
...
Creating deployment package.
Updating policy for IAM role: sagemaker-lambda-dev
Updating lambda function: sagemaker-lambda-dev
Updating rest API
Resources deployed:
  - Lambda ARN: arn:aws:lambda:xxx
  - Rest API URL: https://yyy.execute-api.ap-northeast-2.amazonaws.com/api/
```

[Postman](https://www.getpostman.com/) 으로 테스트 해봅니다. [MNIST as .jpg](https://www.kaggle.com/scolianni/mnistasjpg) 에서 다운로드 받은 jpg 이미지 중 하나를 골라봅시다. multipart/form-data 에 `img` 를 key 로 지정하고, 이전 단계에서 확인한 Rest API URL 에 때려봅니다.

<p align="center">
  <img src="https://raw.githubusercontent.com/chang12/chang12.github.io/master/images/2019-07-21-pic3-result.png" alt="2019-07-21-pic3-result.png"/>
</p>

추론된 label 값이 올바른 것을 확인할 수 있습니다.

## 레퍼런스

* [Get Started - Amazon SageMaker - AWS Documentation](https://docs.aws.amazon.com/sagemaker/latest/dg/gs.html)
* [chalice - Binary Content](https://chalice.readthedocs.io/en/latest/topics/views.html#binary-content)
