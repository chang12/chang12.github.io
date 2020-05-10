---
layout: post
title: "API Gateway 로 Kinesis 를 위한 HTTP Proxy 만들기"
tags: []
---

[AppsFlyer](https://www.appsflyer.com/) 라는 SaaS 서비스의 데이터를 받아와서 처리하고 싶은 니즈가 생겼습니다. AppsFlyer 는 Event 가 발생할 때 마다 사용자가 등록한 HTTP Endpoint 로 데이터를 쏴주는 [Push API](https://support.appsflyer.com/hc/en-us/articles/207034356-Push-API-V2-0-real-time-attribution-event-reporting) 라는 기능을 제공합니다. 이를 사용해서 HTTP Endpoint 로 데이터를 받고, 후속 처리를 위해 Kinesis Data Stream 에 넣으면 되겠습니다. API Gateway 로 HTTP Endpoint 를 만들고 Kinesis 에 연동할 수 있어 관련 내용을 정리해봤습니다. 

## API 생성

API Gateway Console 에서 API 를 생성합니다. Create API 클릭 -> REST API 항목의 Build 를 클릭합니다.

- **Choose the protocol** : REST
- **Create new API** : New API
- **API name** : KinesisProxy
- **Endpoint Type** : Regional

## IAM Role 생성

Method 를 생성하기에 앞서, API Gateway 의 Method 가 사용할 IAM Role 을 생성합니다. Console 에서 Create role 를 클릭합니다.

- **Select type of trusted entity** : AWS service
- **Choose a use case** : API Gateway
- **Role name** : api-gateway-kinesis-proxy

API Gateway 에서 Kinesis 에 접근할 수 있도록, 생성된 Role 에 **AmazonKinesisFullAccess** policy 도 attach 합니다.

## Method 생성

REST API 를 생성하면 자동으로 `/` 경로에 대한 Resource 가 만들어집니다. 해당 Resource 에 대해 POST method 를 생성합니다. [Tutorial: Create a REST API as an Amazon Kinesis proxy in API Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/integrating-api-with-aws-services-kinesis.html) 문서를 참고했습니다.

- **Integration type** : AWS Service 
- **AWS Region** : 원하는 리전 (e.g. ap-northeast-2)
- **AWS Service** : Kinesis
- **HTTP method** : POST
- **Action Type** : Use action name
- **Action** : PutRecord
- **Execution role** : 위에서 생성한 **api-gateway-kinesis-proxy role** 의 arn

## Kinesis Data Stream 생성

API Gateway 로 데이터를 받아 넣어줄 Kinesis Data Stream 을 생성합니다. Console 에서 생성합니다.

- **Data stream name** : api-gateway-kinesis-proxy
- **Number of open shards** : 1 (최소값)

## Method 의 Integration Request 설정

### HTTP Headers

Kinesis 에서 요청을 수용할 수 있도록 HTTP Header 를 추가합니다.

- **Content-Type** : 'application/x-amz-json-1.1'

### Mapping Templates

Kinesis 의 PutRecord API 는 3개의 argument 를 필수로 요구합니다.

- **StreamName** : (앞서 생성한) api-gateway-kinesis-proxy
- **Data** : Appsflyer 의 Push API 는 Event 데이터를 JSON 으로 HTTP Request Body 에 넣어 요청해줍니다. [Push API V2.0 real-time attribution event reporting 문서의 Message structure and unique fields](https://support.appsflyer.com/hc/en-us/articles/207034356-Push-API-V2-0-real-time-attribution-event-reporting#push-api-20-message-structure-and-unique-fields) 에서 데이터 샘플을 확인할 수 있습니다. 그러니 HTTP Request Body 를 그대로 **Data** 로 넘기면 됩니다. 이때 Kinesis 는 Data 가 base64 encoded 되어있음을 가정하므로, mapping template 의 기능을 활용하여 encoding 해줍니다.
- **PartitionKey** : 적절한 랜덤값을 생성해서 넣어주고 싶습니다. context 의 requestId 를 쓰면 적절합니다.

이들 3개 argument 를 위한 mapping template 을 **Content-Type : application/json** 에 대해 추가합니다.

```json
{
  "StreamName": "api-gateway-kinesis-proxy",
  "Data": "$util.base64Encode($input.body)",
  "PartitionKey": "$context.requestId"
}
```

## 테스트

API Gateway Console 에서 샘플 데이터로 테스트해봅니다.

```json
{
  "app_id": "com.sokoloff06.sdktest",
  "event_name": "af_cross_promotion",
  "event_time": "2020-01-19 10:57:26.038",
  "idfa": null,
  "advertising_id": "cb654aa6-8026-4633-bfd1-de619896fd6a"
}
```

정상적으로 처리되고 Response Body 가 내려옵니다.

```json
{
  "SequenceNumber": "49606871029430345150602562697147520083562131474029215746",
  "ShardId": "shardId-000000000000"
}
```

Logs 를 보니 데이터는 base64 encoding 되어 Kinesis 에 요청됩니다. 

```
ewogICJhcHBfaWQiOiAiY29tLnNva29sb2ZmMDYuc2RrdGVzdCIsCiAgImV2ZW50X25hbWUiOiAiYWZfY3Jvc3NfcHJvbW90aW9uIiwKICAiZXZlbnRfdGltZSI6ICIyMDIwLTAxLTE5IDEwOjU3OjI2LjAzOCIsCiAgImlkZmEiOiBudWxsLAogICJhZHZlcnRpc2luZ19pZCI6ICJjYjY1NGFhNi04MDI2LTQ2MzMtYmZkMS1kZTYxOTg5NmZkNmEiCn0=
```

## 레퍼런스

- [Tutorial: Create a REST API as an Amazon Kinesis proxy in API Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/integrating-api-with-aws-services-kinesis.html)
- [Appsflyer : Message structure and unique fields](https://support.appsflyer.com/hc/en-us/articles/207034356-Push-API-V2-0-real-time-attribution-event-reporting#push-api-20-message-structure-and-unique-fields)
- [API Gateway mapping template and access logging variable reference](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-mapping-template-reference.html)