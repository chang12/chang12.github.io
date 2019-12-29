---
layout: post
title: "Fluent Bit 로 Kubernetes 에 배포된 어플리케이션 로그 수집하기"
tags: [Kubernetes, AWS, Fluent Bit]
---

[이전 글에서](https://chang12.github.io/eks-flask/) 서버 로그 수집에 Fluent Bit 를 도입할 수 있을지 탐색할 계획이라고 얘기하며, EKS 로 클러스터를 띄우고 간단한 Flask 어플리케이션을 배포해봤습니다. 이번에는 Fluent Bit 으로 로그를 수집하는 간단한 데모를 진행하고 관련 내용을 정리해봤습니다.

현재는 Spring Boot 어플리케이션의 Logback 설정에 [KinesisAppender](https://github.com/guardian/kinesis-logback-appender) 를 사용해서 직접 Kinesis Data Stream 에 로그를 넣고, 이를 Firehose 의 Source 로 하여 Destination S3 에 파일로 쌓고 있습니다. 이와 유사하게 Pod 의 Container 의 Process 가 stdout 으로 출력한 로그를 Fluent Bit 을 통해 Firehose 로 수집한 뒤 S3 에 파일로 쌓는 걸 목표로 삼았습니다. 

관련하여 거의 비슷한 내용을 다룬 AWS 블로그의 [Centralized Container Logging with Fluent Bit](https://aws.amazon.com/blogs/opensource/centralized-container-logging-fluent-bit/) 글이 있어 많이 참고했습니다.

## AWS 리소스 준비

### S3 Bucket

Firehose delivery stream 이 로그를 쌓을 S3 Bucket 을 생성합니다. `eks-fluent-bit-study` 정도의 이름으로 만들었습니다.

### Kinesis Firehose delivery stream

Source 는 Direct Put + Destination 은 방금 생성한 S3 Bucket 으로 해서 delivery stream 을 생성합니다. Buffer size / interval 모두 최소값으로 해서 나중에 S3 에 쌓인 파일을 가능한 빠르게 확인할 수 있게 합니다.

- **Delivery stream name** : eks-fluent-bit-study
- **Source** : Direct PUT
- **Data transformation** : Disabled
- **Record format conversion** : Disabled
- **Destination** : Amazon S3
- **S3 bucket** : eks-fluent-bit-study
- **Buffer size** : 1MB (최소값)
- **Buffer interval** : 60 seconds (최소값)
- **S3 compression** : Disabled
- **S3 encryption** : Disabled
- **Error logging** : Enabled
- **IAM role** : AWS 콘솔에서 Create new or choose 클릭 -> firehose\_delivery\_role 로 생성한 뒤 선택

### EKS Node Group

Node 의 File System 에 쌓이는 로그 파일의 경로나 내용을 볼 수 있으면 좋으니, `--ssh-public-key` 옵션으로 Key Pair 를 지정해서, EC2 인스턴스에 SSH 로 접속할 수 있게 합니다.

```
eksctl create nodegroup \
	--cluster kubernetes-study \
	--name python-app \
	--node-type t3.medium \ 
	--nodes 2 \ 
	--nodes-min 2 \ 
	--nodes-max 2 \
	--ssh-access \ 
	--ssh-public-key <key-pair-name> \
	--managed
```

이때 EKS 가 알아서 IAM Role 을 생성하고 Node Group 의 EC2 에 지정합니다. 추후 Node 에서 Daemon Set 으로 실행되는 Fluent Bit Pod 에서 Firehose delivery stream 에 record 를 put 할 수 있도록 적절한 Policy 를 추가합니다. 데모니까 간편하게 `AmazonKinesisFirehoseFullAccess` Policy 를 줍니다.

## 로그 발생 시키는 Process 생성

주기적으로 stdout 으로 로그를 출력하는 간단한 Python Process 를 생성합니다.

```python
import json
import time

if __name__ == '__main__':
    while True:
        d = {'name': 'fakenerd', 'time_ms': int(time.time() * 1000)}
        print(json.dumps(d))
        time.sleep(1)
```

Dockerfile 도 작성하고, build 후 ECR 에 push 합니다.

```docker
FROM python:3.7.5-slim

WORKDIR /app
ADD . /app

ENTRYPOINT ["python", "-u", "app.py"]
```

```
docker build . -t <aws-account-id>.dkr.ecr.ap-northeast-2.amazonaws.com/kubernetes-study
docker push <aws-account-id>.dkr.ecr.ap-northeast-2.amazonaws.com/kubernetes-study
```

Deployment 를 YAML 파일로 기술한 뒤, kubectl 로 반영합니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: python-app
  labels:
    name: python-app
spec:
  replicas: 2
  selector:
    matchLabels:
      name: python-app
  template:
    metadata:
      name: python-app
      labels:
        name: python-app
    spec:
      containers:
        - name: python-app
          image: <aws-account-id>.dkr.ecr.ap-northeast-2.amazonaws.com/kubernetes-study:latest
          resources:
            requests:
              memory: 256Mi
            limits:
              memory: 512Mi
```

```
kubectl apply -f eks-fluent-bit-deployment.yaml
```

Pod 이 정상적으로 떴는지 확인합니다.

```
$ kubectl get pods
NAME                          READY   STATUS    RESTARTS   AGE
python-app-7cd4ffbdc7-8r2ck   1/1     Running   0          6s
python-app-7cd4ffbdc7-mfz7g   1/1     Running   0          8s
```

## 로그 파일 확인

Node Group 생성할 때 Key Pair 를 지정했으므로 SSH 로 Node 에 접속할 수 있습니다.

```
ssh -i <path-to-private-key> ec2-user@<public-dns-of-node>
```

`/var/log/containers` 경로 아래 `python-app` 으로 시작하는 파일을 확인할 수 있습니다. 출력해보면 Pod 의 Process 에서 stdout 으로 출력한 로그를 담고 있습니다.

```
$ tail -f /var/log/containers/python-app-xxx.log
{"log":"{\"name\": \"fakenerd\", \"time_ms\": 1577521193704}\n","stream":"stdout","time":"2019-12-28T08:19:53.704659587Z"}
{"log":"{\"name\": \"fakenerd\", \"time_ms\": 1577521194705}\n","stream":"stdout","time":"2019-12-28T08:19:54.705860842Z"}
...
```

SSH 로 접속하지 않고 kubectl 로 확인할 수도 있습니다. 출력에 적혀있듯이 Deployment 의 Pod 2개 중 하나를 골라서 보여줍니다. [Logging Architecture](https://kubernetes.io/docs/concepts/cluster-administration/logging/) 문서에 따르면 결국 이 출력도 위에서 확인한 로그 파일을 읽어 보여준 것입니다.

> When you run kubectl logs as in the basic logging example, the kubelet on the node handles the request and reads directly from the **log file**, returning the contents in the response.

```
$ kubectl logs -f deployment/python-app --since 1s
Found 2 pods, using pod/python-app-95666c5bb-tl8fr
{"name": "fakenerd", "time_ms": 1577522673575}
{"name": "fakenerd", "time_ms": 1577522674576}
{"name": "fakenerd", "time_ms": 1577522675577}
...
```

Container 의 Process 가 stdout 으로 출력한 것들이 이렇게 Node 에 파일로 쌓일 수 있었던 건 [Docker daemon 의 logging driver](https://docs.docker.com/config/containers/logging/configure/) 덕분입니다. Docker 는 여러 logging driver 를 지원하는데 default 로는`json-file` 를 사용합니다. 이는 Node 에서 `daemon.json` 을 출력해서 확인할 수 있습니다.

```
$ cat /etc/docker/daemon.json
{
  "bridge": "none",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "10"
  },
  "live-restore": true,
  "max-concurrent-downloads": 10
}
```

[JSON File logging driver](https://docs.docker.com/config/containers/logging/json-file/) 문서에서 소개하듯이 (그리고 위에서 tail 로 조회했을 때도 그러했듯이) `json-file` logging driver 는 Process 가 stdout / stderr 로 출력한 걸 `log` 라는 key 로, stdout / stderr 여부를 `stream` key 로, timestamp 를 `time` key 로 json 을 만들어 파일에 적습니다.

## Fluent Bit 관련 Kubernetes Objects 생성

앞서 언급한 [Centralized Container Logging with Fluent Bit](https://aws.amazon.com/blogs/opensource/centralized-container-logging-fluent-bit/) 글에서 Fluent Bit 을 어떻게 배포해야 하는지 간단히 소개합니다. 블로그 글에서도 그러했고, 좀 더 간단할 것 같아서, DaemonSet 으로 배포합니다. 앞서 언급한 [Logging Architecture](https://kubernetes.io/docs/concepts/cluster-administration/logging/) 문서를 읽어보면 더 다양한 방법들에 대해 알 수 있습니다.

> Next comes the routing component: this is Fluent Bit. It takes care of reading logs from all sources and routing log records to various destinations, also known as log sinks. This routing component needs to run somewhere, **for example as a sidecar in a Kubernetes pod / ECS task, or as a host-level daemon set.**

### DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentbit
  labels:
    app.kubernetes.io/name: fluentbit
spec:
  selector:
    matchLabels:
      name: fluentbit
  template:
    metadata:
      labels:
        name: fluentbit
    spec:
      containers:
      - name: aws-for-fluent-bit
        image: amazon/aws-for-fluent-bit:2.1.0
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: fluent-bit-config
          mountPath: /fluent-bit/etc/
        - name: mnt
          mountPath: /mnt
          readOnly: true
        resources:
          limits:
            memory: 500Mi
          requests:
            cpu: 500m
            memory: 100Mi
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: fluent-bit-config
        configMap:
          name: fluent-bit-config
      - name: mnt
        hostPath:
          path: /mnt
```

- AWS 블로그 글을 따라서 [amazon/aws-for-fluent-bit](https://github.com/aws/aws-for-fluent-bit) 이미지를 사용합니다. [AWS 서비스들에 대한 Output Plugin 들을](https://github.com/aws/aws-for-fluent-bit#plugins) 포함하여 빌드한 이미지라 편리합니다. 덕분에 `firehose` output plugin 을 사용할 수 있습니다. 블로그 글에서는 1.2.0 버전을 쓰지만, 이 글을 쓰는 시점에서 최신인 2.1.0 버전으로 올렸습니다. 
- 앞서 Node 에 SSH 로 접속해서 `/var/log/containers` 경로 아래 로그 파일들을 조회했습니다. Fluent Bit 이 해당 경로를 조회할 수 있도록 `hostPath` 를 마운트 합니다. ConfigMap 으로 관리하는 Fluent Bit 설정 파일들도 (`fluent-bit-config`) 마운트 합니다.

### ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  labels:
    app.kubernetes.io/name: fluentbit
data:
  fluent-bit.conf: |
    [SERVICE]
        Parsers_File      parsers.conf
    [INPUT]
        Name              tail
        Tag               kube.*
        Parser            docker
        Path              /var/log/containers/python-app-*.log
        DB                /var/log/flb_kube.db
        Mem_Buf_Limit     5MB
        Skip_Long_Lines   On
        Refresh_Interval  10
    [FILTER]
        Name              record_modifier
        Match             kube.*
        Whitelist_key     log
    [FILTER]
        Name              nest
        Match             kube.*
        Operation         lift
        Nested_under      log
    [OUTPUT]
        Name              firehose
        Match             kube.*
        region            ap-northeast-2
        delivery_stream   eks-fluent-bit-study
  parsers.conf: |
    [PARSER]
        Name              docker
        Format            json
        Time_Key          time
        Time_Format       %Y-%m-%dT%H:%M:%S.%L
        Time_Keep         On
        # Command       | Decoder   | Field | Optional Action   |
        # ==============|===========|=======|===================|
        Decode_Field_As   json        log
```

Fluent Bit 설정 파일에 대해서는 뒤에서 자세히 얘기해보겠습니다.

위에 적힌 YAML 파일대로 배포하면 각각의 Node 에 대한 DaemonSet 의 Pod 으로 Fluent Bit Process 가 실행되고 Firehose 에 record 를 넣기 시작합니다.

```
$ kubectl apply -f eks-fluent-bit-configmap.yaml
$ kubectl apply -f eks-fluent-bit-daemonset.yaml
```

Firehose 의 Buffer interval 을 60초로 했으므로 1분여가 지나면 S3 에 파일이 쌓이기 시작합니다. S3 에 쌓인 파일을 조회해보면 앞서 배포한 Python Process 가 stdout 으로 내보낸 dict 로그 데이터임을 확인할 수 있습니다.

```
{"name":"fakenerd","time_ms":1577622109635}
```

## Fluent Bit 설정

Fluent Bit 가 로그 파일을 읽은 뒤 쓸 Parser 가 필요합니다. [Parsers](https://docs.fluentbit.io/manual/parser) 문서를 보면 Fluent Bit 이 제공하는 pre-configured parsers 목록을 볼 수 있는데, 그 중에서 `json-file` logging driver 가 만든 로그 파일에 쓸 수 있는 `docker` parser 가 있습니다. [fluent/fluent-bit GitHub](https://github.com/fluent/fluent-bit/blob/262267287641d6d54bb62f0a0ada1c76a5bd40b4/conf/parsers.conf#L40-L53) 에서 설정을 확인할 수 있습니다.

```
[PARSER]
    Name              docker
    Format            json
    Time_Key          time
    Time_Format       %Y-%m-%dT%H:%M:%S.%L
    Time_Keep         On
```

INPUT, OUTPUT Plugin 은 일단 앞서 언급한 [Centralized Container Logging with Fluent Bit](https://aws.amazon.com/blogs/opensource/centralized-container-logging-fluent-bit/) 의 내용을 그대로 옮겨오고 파일 경로와 delivery stream 값만 바꿔줍니다.

```
[INPUT]
    Name              tail
    Tag               kube.*
    Parser            docker
    Path              /var/log/containers/python-app-*.log
    DB                /var/log/flb_kube.db
    Mem_Buf_Limit     5MB
    Skip_Long_Lines   On
    Refresh_Interval  10
[OUTPUT]
    Name              firehose
    Match             kube.*
    region            ap-northeast-2
    delivery_stream   eks-fluent-bit-study
```

여기까지 설정하고 배포한 뒤 S3 에 쌓인 파일을 조회하면 아래와 같습니다.

```
{"log":"{\"name\": \"fakenerd\", \"time_ms\": 1577613380890}\n","stream":"stdout","time":"2019-12-29T09:56:20.890502347Z"}
```

`log` key 의 로그 문자열이 JSON 이 아니라 encoded string 으로 처리된 것을 볼 수 있습니다. Fluent Bit 에서는 이러한 encoded 데이터를 처리할 수 있도록 [Decoders](https://docs.fluentbit.io/manual/parser/decoder) 를 제공합니다. 이번에는 `json` decoder 를 사용하면 되겠습니다. 기존의 `log` key 의 encoded string 은 유지할 필요 없으니 `Decode_Field_As` type 으로 decoder 를 사용합니다.

```
[PARSER]
    Name              docker
    Format            json
    Time_Key          time
    Time_Format       %Y-%m-%dT%H:%M:%S.%L
    Time_Keep         On
    # Command       | Decoder   | Field | Optional Action   |
    # ==============|===========|=======|===================|
    Decode_Field_As   json        log
```

다시 배포한 뒤 S3 에 쌓인 파일을 조회하면 아래와 같습니다. `log` key 의 값이 JSON 으로 처리된 것을 확인할 수 있습니다.

```
{"log":{"name":"fakenerd","time_ms":1577615124809},"stream":"stdout","time":"2019-12-29T10:25:24.810156229Z"}
```

`stream`, `time` key 는 필요 없으니 [Record Modifier Filter](https://docs.fluentbit.io/manual/filter/record_modifier) 를 추가해서 `log` key 만 취합니다.

```
[FILTER]
    Name              record_modifier
    Match             kube.*
    Whitelist_key     log
```

```
{"log":{"name":"fakenerd","time_ms":1577621729942}}
```

마지막으로 [Nest Filter 의 lift operation](https://docs.fluentbit.io/manual/filter/nest#configuration-parameters) 을 써서 `log` key 의 값으로 들어가있는 로그 데이터를 한 겹 밖으로 꺼냅니다.

```
[FILTER]
    Name              nest
    Match             kube.*
    Operation         lift
    Nested_under      log
```

```
{"name":"fakenerd","time_ms":1577622109635}
```

이렇게 완성한 Fluent Bit 설정 파일을, 앞서 ConfigMap 을 선언한 YAML 파일의 `fluent-bit.conf` 와 `parsers.conf` 에 적었었습니다.

### Kubernetes Filter

앞에서 한 것처럼 `docker` parser 를 정의할 때 `Decode_Field_As` 를 사용해서 encoded string 을 JSON 으로 디코딩하는 대신, [Kubernetes Filter](https://docs.fluentbit.io/manual/filter/kubernetes) 를 쓰는 방법도 있습니다. Kubernetes Filter 를 쓰면 Kubernetes metadata 를 추가로 매달 수 있다는 장점도 있습니다. 하지만 이번 데모에서는 필요하지 않았기에, 담백하게 `docker` parser 로 디코딩하는 방법을 택했었습니다. 

Kubernetes Filter 를 쓸 경우 `Merge_Log` 를 Off -> On 하면 `log` key 값을 JSON 으로 디코딩 합니다.

> When enabled, it checks if the `log` field content is a JSON string map, if so, it append the map fields as part of the log structure.

`Keep_Log` 를 On -> Off 해서 기존의 encoded string 의 `log` key 는 제거하고, `Merge_Log_Key` 를 log 로 줘서, JSON 으로 디코딩한 값을 새로운 `log` key 의 값으로 넣습니다. 이어지는 `record_modifier` 와 `nest` 설정은 앞에서와 동일합니다.

```
[FILTER]
    Name              kubernetes
    Match             kube.*
    Merge_Log         On
    Keep_Log          Off
    Merge_Log_Key     log
[FILTER]
    Name              record_modifier
    Match             kube.*
    Whitelist_key     log
[FILTER]
    Name              nest
    Match             kube.*
    Operation         lift
    Nested_under      log    
```

동일한 결과를 확인할 수 있습니다.

```
{"name":"fakenerd","time_ms":1577622955572}
```

## 레퍼런스

- [AWS Open Source Blog : Centralized Container Logging with Fluent Bit](https://aws.amazon.com/blogs/opensource/centralized-container-logging-fluent-bit/)
- [kubernetes : Logging Architecture](https://kubernetes.io/docs/concepts/cluster-administration/logging/)
- [docker docs : Configure logging drivers](https://docs.docker.com/config/containers/logging/configure/)
- [docker docs : JSON File logging driver](https://docs.docker.com/config/containers/logging/json-file/)
- [aws/aws-for-fluent-bit](https://github.com/aws/aws-for-fluent-bit)
- [Fluent Bit: Official Manual - Parsers](https://docs.fluentbit.io/manual/parser)
- [fluent/fluent-bit GitHub : fluent-bit/conf/parsers.conf](https://github.com/fluent/fluent-bit/blob/262267287641d6d54bb62f0a0ada1c76a5bd40b4/conf/parsers.conf#L40-L53)
- [Fluent Bit: Official Manual - Decoders](https://docs.fluentbit.io/manual/parser/decoder)
- [Fluent Bit: Official Manual - Record Modifier](https://docs.fluentbit.io/manual/filter/record_modifier)
- [Fluent Bit: Official Manual - Nest](https://docs.fluentbit.io/manual/filter/nest#configuration-parameters)
- [Fluent Bit: Official Manual - Kubernetes](https://docs.fluentbit.io/manual/filter/kubernetes)
