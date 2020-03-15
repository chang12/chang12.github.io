---
layout: post
title: "Cloud Run 과 Cloud Scheduler 로 주기적인 작업 실행"
tags: [GCP]
---

글또 4기 활동을 하면서는, 운영진 팀 2개 중 운영 자동화 팀의 일원으로도 활동합니다. 글 제출/피드백 여부 확인, 리뷰어 지정 등의 글또 운영 업무를 자동화 하는 [genie](https://github.com/geultto/genie) 프로젝트를 관리하고 개선하는 것이 운영 자동화 팀의 역할입니다. 그 일환으로 주기적으로 Slack API 로 데이터를 받아와서 (글또 활동은 Slack 에서 이뤄짐) 처리하는 작업을 실행해야 합니다.

찾아보니 GCP 의 서비스들 중 Cloud Run 으로 작업을 배포하고, Cloud Scheduler 로 주기적으로 실행하면 될 것 같습니다. [Running services on a schedule](https://cloud.google.com/run/docs/triggering/using-scheduler) 에서 개괄적인 내용을 확인할 수 있습니다.

## IAM 에서 Service Account 생성

Cloud Scheduler 가 Cloud Run 의 Service 를 호출할 때 인증에 쓸 IAM Service Account 가 필요합니다.

```
gcloud iam service-accounts create fakenerd-cloud-scheduler
```

만들어진 Service Account 의 이메일을 확인합니다. 이후 Service 에 Policy Binding 추가할 때 사용합니다.

```
gcloud iam service-accounts list | grep fakenerd-cloud-scheduler
```

## Cloud Run Service 생성

[Cloud Run 페이지를](https://cloud.google.com/run) 보면 **'Cloud Run is a fully managed compute platform that automatically scales your stateless containers.'** 라고 소개합니다. 원하는 작업을 코드로 기술하고 Dockerfile 만 작성해주면 Container 를 운영하는 기반 작업은 Cloud Run 이 해줍니다.

[Quickstart: Deploy a Prebuilt Sample Container](https://cloud.google.com/run/docs/quickstarts/prebuilt-deploy) 문서의 내용을 gcloud 명령어로 옮긴 아래 명령어를 실행해서 Cloud Run 에 Service 를 배포할 수 있습니다. GCP 에서 준비해준 gcr.io/cloudrun/hello image 에 대해서는 [GoogleCloudPlatform/cloud-run-hello](https://github.com/GoogleCloudPlatform/cloud-run-hello) repo 에서 코드를 확인할 수 있습니다.

```
gcloud run deploy prebuilt \
    --image gcr.io/cloudrun/hello \
    --platform managed \
    --no-allow-unauthenticated \
    --region us-central1
```

배포가 완료되면 Service 의 URL 을 확인할 수 있습니다. 이후 Cloud Scheduler Job 을 만들 때 필요합니다.

```
Deploying container to Cloud Run service [prebuilt] in project [...] region [us-central1]
✓ Deploying... Done.
  ✓ Creating Revision...
  ✓ Routing traffic...
  ✓ Setting IAM Policy...
Done.
Service [prebuilt] revision [prebuilt-...] has been deployed and is serving 100 percent of traffic at https://prebuilt-...-uc.a.run.app
```

`--no-allow-unauthenticated` 로 Service 를 만들었으므로 호출하기 위해서는 인증이 필요합니다. 앞서 생성한 Service Account 명의로 (?) Service 를 호출할 수 있도록 IAM Policy Binding 을 추가합니다.

```
gcloud run services add-iam-policy-binding prebuilt \
    --member serviceAccount:<앞에서_만든_SERVICE_ACCOUNT_의_EMAIL> \
    --role roles/run.invoker \
    --platform managed \
    --region us-central1
```

## Cloud Scheduler Job 생성

[Cloud Scheduler 페이지를](https://cloud.google.com/scheduler) 보면 **'Cloud Scheduler is a fully managed enterprise-grade cron job scheduler.'** 라고 소개합니다. [Cloud Scheduler overview](https://cloud.google.com/scheduler/docs) 문서를 보면 HTTP/S endpoints, Pub/Sub topics, App Engine applications 으로 3가지 Target 이 가능합니다. Cloud Run Service 를 호출하는 것이 목적이므로 HTTP/S endpoints 를 Target 으로 Job 을 만듭니다. Job 이름, 원하는 cron 주기, 호출할 Cloud Run Service 의 Endpoint, 인증에 쓸 Service Account 를 적어주면 됩니다. 빠른 테스트를 위해 최소 간격인 매분 실행으로 설정합니다.

```
gcloud scheduler jobs create http trigger-cloud-run \
    --schedule '* * * * *' \
    --uri <앞에서_배포한_CLOUD_RUN_SERVICE_의_URL> \
    --http-method get \
    --oidc-service-account-email <앞에서_만든_SERVICE_ACCOUNT_의_EMAIL> \
    --oidc-token-audience <앞에서_배포한_CLOUD_RUN_SERVICE_의_URL>
```

## 확인

Console 에서 Service 의 LOGS 탭을 보면, HTTP 호출이 성공한 것을 확인할 수 있습니다.

![2020-03-15-cloud-run-logs.png](https://raw.githubusercontent.com/chang12/chang12.github.io/88ffd9973f513063a85219a4823fcd86af9b19e0/images/2020-03-15-cloud-run-logs.png)

## 레퍼런스

- [geultto/genie](https://github.com/geultto/genie)
- [Cloud Run 문서 중 : Running services on a schedule](https://cloud.google.com/run/docs/triggering/using-scheduler)
- [Cloud Run 서비스 소개 페이지](https://cloud.google.com/run)
- [Cloud Run 문서 중 : Quickstart: Deploy a Prebuilt Sample Container](https://cloud.google.com/run/docs/quickstarts/prebuilt-deploy)
- [GoogleCloudPlatform/cloud-run-hello](https://github.com/GoogleCloudPlatform/cloud-run-hello)
- [Cloud Scheduler 서비스 소개 페이지](https://cloud.google.com/scheduler)
- [Cloud Scheduler 문서 중 : Cloud Scheduler overview](https://cloud.google.com/scheduler/docs)
