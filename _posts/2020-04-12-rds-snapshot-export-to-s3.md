---
layout: post
title: "Aurora MySQL Snapshot 을 S3 에 Parquet 로 Export"
tags: [AWS]
---

[Amazon RDS/Aurora Snapshot 을 S3 에 Apache Parquet 로 Export 하는 기능이 지난 1월에](https://aws.amazon.com/about-aws/whats-new/2020/01/announcing-amazon-relational-database-service-snapshot-export-to-s3/) 출시되었습니다. Aurora MySQL 에 대해 직접 실습해보고 내용을 정리해봤습니다. 

아쉽게도 아직 서울 리전에는 출시되지 않아, 도쿄 리전에서 실습했습니다. [Exporting DB Snapshot Data to Amazon S3](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/USER_ExportSnapshot.html) 문서의 내용을 많이 참고했습니다. Pricing 관련 [Amazon Aurora Pricing 의 Snapshot Export](https://aws.amazon.com/rds/aurora/pricing/?pg=pr&loc=1) 를 보니 도쿄 리전은 **Charge per GB of snapshot size: $0.012** 입니다.


## AWS CLI 인증

AWS CLI 를 사용해서 각종 AWS Resource 를 만들기 위해 인증이 필요합니다. 간편하게 콘솔에서 IAM User 를 만들고 **AdministratorAccess** Policy 를 부여합니다. [Configuration and Credential File Settings](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html) 를 참고해서 config/credential 파일을 작성한 뒤 AWS CLI 명령어에 `--profile` option 을 줘서 인증합니다. 

`~/.aws/credentials`

```
[fakenerd]
aws_access_key_id=<ACCESS_KEY>
aws_secret_access_key=<SECRET_KEY>
```

`~/.aws/config`

```
[profile fakenerd]
region = ap-northeast-1
output = json
```

## Security Group 생성

Default VPC 에 Security Group 을 생성합니다.

```
aws ec2 create-security-group \
    --description 'Allows Aurora MySQL from outside.' \
    --group-name rds-snapshot-export-to-s3
```

개인 랩탑에서 접속할 수 있도록 MySQL 의 기본 Port 인 3306 을 외부에 개방합니다.

```
aws ec2 authorize-security-group-ingress \
    --group-id <SECURITY_GROUP_ID_ABOVE> \
    --protocol tcp \
    --port 3306 \
    --cidr 0.0.0.0/0
```

## KMS Key 생성

Aurora DB Cluster 를 띄울 때와 Export Task 를 시작할 때 쓸 KMS Key 를 생성합니다. 생성된 Key 의 ID 를 뒤에서 사용합니다.

```
aws kms create-key
```

## Aurora DB Cluster 생성

- RDS 콘솔에서는 Cluster 를 생성하면 자동으로 Primary Instance 를 생성해줬습니다. 하지만 AWS CLI 를 쓸 때는 Cluster 를 만들고 명시적으로 Primary Instance 를 생성해줘야 합니다.
- 이 글이 쓰인 시점에는 MySQL 5.7 compatible 한 Aurora 버전 중 2.04.4, 2.04.5, 2.04.6 가 Snapshot Export 기능을 제공합니다. 그 중 가장 최신인 **2.04.6** 을 선택했습니다.
- 실습용이니 가장 작은 **db.t3.small** (2 vCPUs/2 GiB RAM/시간당 $0.063) 타입을 선택합니다. 
- 개인 랩탑에서 접속할 수 있도록 **Publicly Accessible** 로 설정합니다.
- 문서에 정확히 언급되어 있지 않아 힘들었던 부분인데, **Storage Encryption 을 꼭 해줘야** 합니다. `--storage-encrypted` 을 명시하지 않으면 Default 는 Not Encrypted 로 Cluster 가 만들어지는데, 그러면 Snapshot 을 만들어도 Not Encrypted 가 됩니다. 그러한 Snapshot 에 대해 Export Task 를 실행하면 성공하긴 하지만 S3 에는 비어있는 Parquet 파일이 만들어지게 됩니다.

```
aws rds create-db-cluster \
    --database-name fakenerd \
    --db-cluster-identifier rds-snapshot-export-to-s3 \
    --vpc-security-group-ids <SECURITY_GROUP_ID_ABOVE> \
    --engine aurora-mysql \
    --engine-version 5.7.mysql_aurora.2.04.6 \
    --master-username admin \
    --master-user-password <PASSWORD> \
    --storage-encrypted \
    --kms-key-id <KMS_KEY_ID_ABOVE>
```

```
aws rds create-db-instance \
    --db-instance-identifier rds-snapshot-export-to-s3-1 \
    --db-instance-class db.t3.small \
    --engine aurora-mysql \
    --publicly-accessible \
    --db-cluster-identifier rds-snapshot-export-to-s3
```

## 데이터베이스에 데이터 Load

[Kaggle 의 Datasets](https://www.kaggle.com/datasets) 중 적당한 걸 찾다가 [Air Pollution in Seoul](https://www.kaggle.com/bappekim/air-pollution-in-seoul) 의 Measurement_summary.csv 를 선택했습니다. Kaggle 웹페이지에서 데이터를 테이블로 보여주니 이후 DDL 작성에 참고합니다.

**mysql-client** 로 Aurora MySQL 에 접속합니다. `--local-infile` option 을 주지 않으면, load data 때 에러가 발생합니다.

```
mysql -h <AURORA_ENDPOINT> -u admin -p --local-infile fakenerd
```

테이블을 생성합니다. CSV 파일의 Header 의 컬럼 이름에는 공백이나 `.` 가 포함된 경우가 있지만, 테이블 만들 때는 없앱니다. 그렇게 하지 않으면 이후 Export Task 를 실행했을 때 **COLUMN\_WITH\_UNSUPPORTED\_CHARACTER\_IN\_NAME** Reason 으로 테이블이 Skip 되거나, 빈 Parquet 파일이 만들어집니다.

```sql
create table measurement_summary (
    measurement_date varchar(32),
    station_code int,
    address varchar(128),
    latitude double,
    longitude double,
    so2 double,
    no2 double,
    o3 double,
    co double,
    pm10 double,
    pm2_5 double
);
```

LOAD DATA 명령어로 Kaggle 에서 다운로드 받은 CSV 파일을 Aurora MySQL 로 올립니다. MySQL 의 [LOAD DATA](https://dev.mysql.com/doc/refman/5.7/en/load-data.html) 를 참고했습니다. 

- CSV 의 첫 줄이 Header 이니 `ignore 1 lines` 로 무시합니다.
- CSV 니까 `fields` 에 `terminated by ','` 를 명시합니다.
- **"19, Jong-ro 35ga-gil, Jongno-gu, Seoul, Republic of Korea"** 같이 Address 값에 포함된 `,` 를 delimiter 로 처리하지 않도록 `fields` 에 `enclosed by '"'` 를 명시합니다.

```
load data
    local
    infile 'Measurement_summary.csv'
    into table measurement_summary
    fields 
        terminated by ','
        enclosed by '"'
    ignore 1 lines;
```

## Snapshot 생성

```
aws rds create-db-cluster-snapshot \
    --db-cluster-snapshot-identifier rds-snapshot-export-to-s3 \
    --db-cluster-identifier rds-snapshot-export-to-s3
```

## S3 Bucket 생성

```
aws s3api create-bucket \
    --bucket rds-snapshot-export-fakenerd \
    --create-bucket-configuration LocationConstraint=ap-northeast-1
```

## IAM Role 생성

S3 Bucket 에 Export 할때 필요한 Permission 들을 포함한 Policy 를 생성합니다.

```
aws iam create-policy \
    --policy-name rds-snapshot-export-to-s3-policy \
    --policy-document '{
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "s3:ListBucket",
                    "s3:GetBucketLocation"
                ],
                "Resource": [
                    "arn:aws:s3:::*"
                ]
            },
            {
                "Effect": "Allow",
                "Action": [
                    "s3:PutObject*",
                    "s3:GetObject*",
                    "s3:DeleteObject*"
                ],
                "Resource": [
                    "arn:aws:s3:::rds-snapshot-export-fakenerd",
                    "arn:aws:s3:::rds-snapshot-export-fakenerd/*"
                ]
            }
        ]
    }'
```

IAM Role 을 생성합니다.

```
aws iam create-role \
    --role-name rds-snapshot-export-to-s3-role \
    --assume-role-policy-document '{
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Principal": {
                    "Service": "export.rds.amazonaws.com"
                },
                "Action": "sts:AssumeRole"
            }
        ] 
    }'
```

Policy 를 Role 에 Attach 합니다.

```
aws iam attach-role-policy \
    --policy-arn <POLICY_ARN_ABOVE> \
    --role-name rds-snapshot-export-to-s3-role
```

## Export Task 실행

```
aws rds start-export-task \
    --export-task-identifier rds-snapshot-export-to-s3-0000 \
    --source-arn <SNAPSHOT_ARN_ABOVE> \
    --s3-bucket-name rds-snapshot-export-fakenerd \
    --iam-role-arn <ROLE_ARN_ABOVE> \
    --kms-key-id <KMS_KEY_ID_ABOVE>
```

Task 의 Status 가 STARTING -> IN_PROGRESS 로 전이하는데 생각보다 긴 (~ 15분) 시간이 걸립니다. [문서를 보면](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/USER_ExportSnapshot.html) **restores and scales the entire database** 한다는데 정확히 어떤 작업인지 모르겠어서 왜 오래 걸리는지 이유를 짐작하지 못했습니다.

> Exporting RDS Snapshots can take a while depending on your database type and size. The export task first restores and scales the entire database before extracting the data to Amazon S3.

IN_PROCESS Status 가 된 후에 정말 Export 하는데는 비교적 짧은 (~ 5분) 시간이 걸렸습니다.

## S3 에 Export 된 Parquet 파일 확인

Export 가 완료되고 나면 `--s3-bucket-name` Bucket 의 `--export-task-identifier` Prefix 에 파일들이 만들어집니다.

```
<BUCKET>/<EXPORT_TASK_IDENTIFIER>
    <DATABASE>
        <DATABASE.TABLE>
            _SUCCESS
            part-xxx.gz.parquet
    export_info_yyy.json
    export_tables_zzz.json
```

export_tables 로 시작하는 JSON 파일을 보면 partitioningInfo 에 numberOfPartitions = numberOfCompletedPartitions = 1 로 적혀있습니다. [문서에서](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/USER_ExportSnapshot.html) partitioning 에 대해 언급한 부분이 생각납니다. Export 한 테이블에 적절한 컬럼이 없어서 별도의 partitioning 없이 단일 Parquet 파일로 Export 되었나 봅니다.

> The time it takes for the export to complete depends on the data stored in the database. For example, tables with **well distributed numeric primary key or index columns** will export the fastest. If tables do not contain **a column suitable for partitioning**, the export becomes a slower single threaded process.

Parquet 는 Binary Format 이라 다운로드 받아도 열어서 내용을 확인하기 어렵습니다. S3 Select 으로 CSV 파일로 저장한 뒤 열어서 Parquet 파일이 잘 만들어졌는지 확인합니다.

```
aws s3api select-object-content \
    --bucket rds-snapshot-export-fakenerd \
    --key <PARQUET_FILE_S3_KEY> \
    --expression "select * from s3object limit 100" \
    --expression-type SQL \
    --input-serialization '{"Parquet": {}}' \
    --output-serialization '{"CSV": {}}' \
    "output.csv"
```

잘 만들어졌습니다.

```
$ head output.csv
2017-01-01 00:00,101,"19, Jong-ro 35ga-gil, Jongno-gu, Seoul, Republic of Korea",37.5720164,127.0050075,0.004,0.059,0.002,1.2,73.0,57.0
2017-01-01 01:00,101,"19, Jong-ro 35ga-gil, Jongno-gu, Seoul, Republic of Korea",37.5720164,127.00ㅇ0075,0.004,0.058,0.002,1.2,71.0,59.0
2017-01-01 02:00,101,"19, Jong-ro 35ga-gil, Jongno-gu, Seoul, Republic of Korea",37.5720164,127.0050075,0.004,0.056,0.002,1.2,70.0,59.0
2017-01-01 03:00,101,"19, Jong-ro 35ga-gil, Jongno-gu, Seoul, Republic of Korea",37.5720164,127.0050075,0.004,0.056,0.002,1.2,70.0,58.0
2017-01-01 04:00,101,"19, Jong-ro 35ga-gil, Jongno-gu, Seoul, Republic of Korea",37.5720164,127.0050075,0.003,0.051,0.002,1.2,69.0,61.0
2017-01-01 05:00,101,"19, Jong-ro 35ga-gil, Jongno-gu, Seoul, Republic of Korea",37.5720164,127.0050075,0.003,0.046,0.002,1.1,70.0,61.0
2017-01-01 06:00,101,"19, Jong-ro 35ga-gil, Jongno-gu, Seoul, Republic of Korea",37.5720164,127.0050075,0.003,0.049,0.002,1.1,66.0,57.0
2017-01-01 07:00,101,"19, Jong-ro 35ga-gil, Jongno-gu, Seoul, Republic of Korea",37.5720164,127.0050075,0.003,0.045,0.002,1.0,71.0,60.0
2017-01-01 08:00,101,"19, Jong-ro 35ga-gil, Jongno-gu, Seoul, Republic of Korea",37.5720164,127.0050075,0.004,0.047,0.002,1.1,72.0,60.0
2017-01-01 09:00,101,"19, Jong-ro 35ga-gil, Jongno-gu, Seoul, Republic of Korea",37.5720164,127.0050075,0.003,0.047,0.002,1.1,74.0,63.0
```

## 마치며

처음 소식을 듣고 잘못 이해하여, 기존에 CSV/TEXT Format 만 가능하던 [SELECT INTO OUTFILE S3](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Integrating.SaveIntoS3.html) 에 Parquet Format 이 추가된 건줄 알고, 기존 Use Case 에 딱 맞는 기능이 나왔다며 기뻐했었습니다. 하지만 실습을 진행하며 그게 아니라 데이터베이스 Snapshot 전체를 Export 하는 거라는 걸 깨닫고 조금은 실망하였습니다. 

하지만 SELECT INTO OUTFILE S3 이 데이터베이스에 부하를 주는데 반해, Export 는 Snapshot 을 쓰므로 부하 없이 데이터베이스의 데이터를 S3 에 저장할 수 있다는 점에서 의의를 지닌다고 생각합니다. Export 에 이어 Glue Crawler/Athena 를 잘 엮어주면, OLTP 데이터를 S3 에 Data Lake 로 만들고, 이를 SQL 로 분석할 수 있는 환경을 서버리스하게 구성할 수 있어 좋아보입니다. 다만 테이블 수준에서는 늘 전체를 다룰 수 밖에 없어 Incremental 하지 못하다는 건 아쉬운 점입니다.

## 레퍼런스

- [Announcing Amazon Relational Database Service (RDS) Snapshot Export to S3](https://aws.amazon.com/about-aws/whats-new/2020/01/announcing-amazon-relational-database-service-snapshot-export-to-s3/)
- [Exporting DB Snapshot Data to Amazon S3](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/USER_ExportSnapshot.html)
- [Amazon Aurora Pricing : Snapshot Export](https://aws.amazon.com/rds/aurora/pricing/?pg=pr&loc=1)
- [Configuration and Credential File Settings](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html)
- [Creating an Amazon Aurora DB Cluster](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.CreateInstance.html)
- [Kaggle : Air Pollution in Seoul](https://www.kaggle.com/bappekim/air-pollution-in-seoul)
- [MySQL 5.7 : 13.2.6 LOAD DATA Statement](https://dev.mysql.com/doc/refman/5.7/en/load-data.html)
- [Saving Data from an Amazon Aurora MySQL DB Cluster into Text Files in an Amazon S3 Bucket](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Integrating.SaveIntoS3.html)
