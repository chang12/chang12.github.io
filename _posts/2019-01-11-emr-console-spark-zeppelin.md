---
layout: post
title: EMR 콘솔에서 Spark, Zeppelin 클러스터 생성
tags: [AWS, Apache Spark]
---

Ad-hoc 분석과 배치 작업에 Apache Spark 를 쓰고 있습니다. 현재 실무에서는 AWS 인프라 위에 직접 Spark 클러스터와 Zeppelin 을 구성해서 사용하고 있습니다. 기존 클러스터 구성이 만족스럽기에 EMR 로 마이그레이션 하진 않겠지만, 언젠가 별도의 인프라를 새롭게 구성해야 한다면 EMR 을 써보고 싶은 마음입니다. 그래서 EMR 에 대해 공부하기 시작했습니다.

## Quick Options 으로 EMR Cluster 생성

EMR 콘솔에서 클러스터를 만들 때 Quick Options 과 Advanced Options 2가지 방식을 제공합니다. Quick Options 에는 함께 자주 사용되는 Hadoop 생태계의 프레임워크들 묶음이 있어서 선택할 수 있어 편리합니다. Spark + Zeppelin 묶음도 준비되어 있으므로, 간단히 몇가지만 설정해주면 몇 분만에 클러스터를 생성할 수 있습니다.

![EMR Quick Options](https://drive.google.com/uc?export=view&id=1KKc8Bn-lZoyznVV935E6ge2zJHeTQ8G1)

클러스터가 정상적으로 만들어지고 **Waiting** (spark 어플리케이션이 제출되길 기다리는 상태) 상태에 이를때 까지 3가지 관찰이 있었습니다.

* **ElasticMapReduce-master**, **ElasticMapReduce-slave** 라는 이름의 security group 을 생성합니다. 각각 spark master / worker EC2 인스턴스에 사용합니다.
* **EMR role**, **EC2 instance profile** 라는 이름의 IAM role 을 생성합니다. 전자는 EMR 이 자체적으로 사용하는 것 같고 (e.g terminate nodes), 후자는 spark master / worker EC2 에 사용합니다.
* 클러스터 상세 화면에서, 해당 클러스터와 동일한 클러스터를 띄울 수 있는 AWS CLI 명령어를 얻을 수 있습니다.

![EMR AWS CLI export](https://drive.google.com/uc?export=view&id=1uLx6VofM3SxukhdRk3L7T7rs4q9fF9hr)

이후 브라우저로 Zeppelin 에 접속하기 위해서 몇 가지 수정이 필요했습니다.

## 클러스터의 Subnet 지정

AWS 에 분석 인프라를 구축할 때 보안을 위해 각종 EC2 머신과 서비스들을 VPC 안에 구성하고, 외부 네트워크와 VPC 사이에 [Bastion 머신을 둬서 SSH 터널링을 통한 접속만 허용하는 방식을](https://blog.scottlowe.org/2015/11/21/using-ssh-bastion-host/) 사용하고 있습니다. 

Quick Options 으로 EMR 클러스터를 생성하면, Default VPC 에 master / worker 를 위한 security group 을 생성하고, 생성한 security group 을 사용해서 master / worker EC2 머신도 Default VPC 에 띄웁니다. 이때 기존 구성 요소들을 띄운 VPC 가 Default VPC 와 달라서 SSH 접속이 안되는 이슈가 있었습니다.

이를 해결하기 위해 AWS CLI export 버튼을 눌러서 획득한 AWS CLI 명령어의 옵션값에 `SubnetId` 를 명시해줍니다.

```sh
aws emr create-cluster \
    ...
    --ec2-attributes '{"KeyName":"...,"SubnetId":"{subnet-id}"}' \
    ...
```

## Security Group 의 inbound rule 수정

EMR 클러스터를 Cluster 모드로 띄우고 어플리케이션에 Zeppelin 을 추가하면, master 머신의 8890 port 에 ZeppelinServer 를 띄웁니다. Bastion 을 통한 SSH 포트 포워딩으로 Zeppelin 에 접속하기 위해 Security Group 수정이 필요했습니다. 또한 Zeppelin 어플리케이션으로 제출한 Spark 프로그램이 RDS 에 접속하기 위해서도 수정이 필요했습니다.

* EMR 이 생성한 **ElasticMapReduce-master** security group 에서 Bastion security group 에게 22 port 에 대한 inbound rule 을 추가합니다.
* RDS security group 에서 **ElasticMapReduce-master**, **ElasticMapReduce-slave** security group 에게 RDBMS port 에 대한 inbound rule 을 추가합니다.

## Zeppelin 접속

위의 2가지를 진행하고 나니, SSH 포트 포워딩과 SOCKS 프록시를 통해 Zeppelin 에 접속할 수 있었습니다. [View Web Interfaces Hosted on Amazon EMR Clusters](https://docs.aws.amazon.com/emr/latest/ManagementGuide/emr-web-interfaces.html) 문서에서 Option 2 로 소개하는 방법입니다. Option 1 에서 소개하는 방법이 방법 자체는 더 간단하지만, (endpoint, port) 마다 포워딩을 추가해야하는 번거로움이 있어서 Option 2 의 방법을 사용하고 있습니다. SOCKS 프록시 설정은 크롬 브라우저의 [FoxyProxy Standard](https://chrome.google.com/webstore/detail/foxyproxy-standard/gcknhkkoolaabfmlnjonogaaifnjlfnp?hl=en) 플러그인을 사용하고 있습니다. 

## 마치며

EMR 의 가호아래, 콘솔에서 간단한 설정만으로 Spark 클러스터와 Zeppelin 을 구성할 수 있었습니다. 

본문에는 적지 않았지만 Quick Options 에서 Spark 묶음을 선택했을 때 Spark History Server 와 Ganglia 라는 것도 띄워주는 데, 처음 접해서 아직 낯설지만 여러모로 유용해보여서 기대됩니다. 기존에 [standalone 모드](https://spark.apache.org/docs/latest/spark-standalone.html) 로 spark 클러스터를 구성했을 때 (별도의 설정을 안해서) 어플리케이션이 종료되고 나면 드라이버 포트로 접속했을 때 보여지는 Web UI 의 데이터가 사라져서 아쉬웠었는데, EMR 이 알아서 Spark History Server 를 구성해줘서 어플리케이션이 종료된 이후에도 각종 모니터링 데이터들을 조회할 수 있었습니다. 또한 기존에는 각 EC2 머신들의 CloudWatch 메트릭들을 봤었는데, Ganglia 가 클러스터 전체의 메트릭들을 잘 묶어서 보여주므로 유익할 듯 합니다.
