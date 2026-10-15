# Amazon S3가 Vercel Marketplace와 v0에서 기본 제공

- 정리 날짜: 2026-10-15
- 원문 게시일: 2026-10-14
- 공식 출처: [AWS News Blog - Amazon S3 is now available on the Vercel Marketplace and v0 by Vercel](https://aws.amazon.com/blogs/aws/amazon-s3-vercel-marketplace-v0/)

## 핵심 내용

Amazon S3, Amazon S3 Tables와 Amazon S3 Vectors를 Vercel Marketplace와 v0 by Vercel에서 직접 생성하고 web 및 AI application에 연결할 수 있게 됐다.

개발자는 AWS Management Console에서 bucket, IAM policy와 credential을 따로 구성하지 않고 Vercel dashboard의 Storage tab에서 AWS를 선택해 필요한 S3 resource를 만들 수 있다. v0에서는 만들고 싶은 application을 자연어로 설명하면 적합한 storage 유형을 provisioning하고 연결 설정과 code까지 생성한다.

세 가지 storage 유형은 서로 다른 workload를 담당한다.

- S3 general purpose bucket: image, video, document와 application file 같은 object storage
- S3 Tables: Apache Iceberg 기반 analytics table storage
- S3 Vectors: RAG, semantic search와 AI agent를 위한 vector storage 및 query

## 왜 중요한가

Vercel에서 application을 만들면서 AWS storage를 사용하려면 이전에는 AWS console에서 resource와 권한을 만들고 access key를 application에 전달해야 했다. 이 과정은 개발 흐름을 끊고 static credential 노출 위험도 만들었다.

새 integration은 provisioning과 connection configuration을 Vercel workflow 안에서 처리한다. resource는 사용자의 AWS account에 생성되므로 AWS-native monitoring, governance와 billing을 유지하면서 Vercel의 빠른 개발 경험을 사용할 수 있다.

기존 Vercel의 Amazon Aurora, DynamoDB와 OpenSearch Serverless integration에 S3 계열이 추가되면서 relational data, key-value data, search, object, open table과 vector storage를 하나의 AWS account와 인증 방식으로 구성할 수 있다.

## 인증 방식

Vercel project를 S3 resource에 연결하면 integration이 다음과 같은 환경 변수를 제공한다.

- `AWS_REGION`
- `AWS_ROLE_ARN`
- `AWS_RESOURCE_ARN`
- `S3_BUCKET_NAME`, `S3_TABLE_BUCKET_NAME` 또는 `S3_VECTOR_BUCKET_NAME`

runtime에서는 `@vercel/functions/oidc` package의 `awsCredentialsProvider`가 short-lived OIDC token을 AWS Security Token Service(AWS STS)에 전달한다. AWS STS는 해당 resource 범위로 제한된 temporary credential을 발급한다.

이 구조는 long-lived access key를 Vercel secret이나 source code에 저장하지 않는다. application은 실행 시점에 필요한 권한을 가진 credential을 짧은 시간 동안만 사용한다.

## Storage 유형별 활용

### S3 General Purpose Bucket

file upload, image와 video, document, data lake와 mobile application asset에 사용할 수 있다. AWS SDK의 `PutObject`, `GetObject`와 `ListObjectsV2` API로 object를 처리한다.

S3는 기본적으로 99.999999999%의 data durability와 99.99% availability를 목표로 설계된 object storage다.

### S3 Tables

Apache Iceberg table을 위한 managed storage다. table maintenance와 compaction을 자동화하고 data volume이 커져도 query 성능을 최적화한다.

Iceberg REST Catalog API와 호환되는 engine에서 query할 수 있다. Vercel Function 안에서 DuckDB를 사용해 table bucket을 ARN으로 attach하고 lightweight analytics query를 실행할 수도 있다.

### S3 Vectors

AI inference, agent, RAG와 semantic search를 위한 vector storage와 similarity query를 제공한다. 별도 vector database 없이 S3에 vector와 metadata를 저장하고 가까운 vector를 검색할 수 있다.

S3 Vectors가 embedding을 생성해 주는 것은 아니다. Amazon Bedrock 또는 선택한 embedding model에서 vector를 생성한 뒤 index에 저장해야 한다.

## v0 by Vercel 사용 방식

v0에서 application 요구 사항을 자연어로 설명하면 workload에 맞는 AWS resource를 선택한다.

- AWS를 사용하는 photo gallery는 general purpose bucket
- PDF 내용을 검색하는 support bot은 S3 Vectors
- clickstream analytics dashboard는 S3 Tables

v0는 resource를 provisioning하고 Vercel project에 연결한 뒤 동작 가능한 application code를 생성한다. 생성된 IAM scope, data model, error handling과 production 설정은 배포 전에 개발자가 검토해야 한다.

## 서비스 및 아키텍처 관점

1. 개발자가 Vercel Marketplace 또는 v0에서 S3 resource 유형과 AWS Region을 선택한다.
2. integration이 연결된 AWS account에 새 bucket 또는 table/vector bucket을 생성한다.
3. Vercel project에 AWS Region, role ARN과 resource identifier가 설정된다.
4. Vercel Function이 실행될 때 OIDC token을 발급받는다.
5. credential provider가 AWS STS의 role assumption을 통해 temporary credential을 얻는다.
6. application이 AWS SDK로 S3, S3 Tables 또는 S3 Vectors API를 호출한다.
7. object metadata는 Aurora에, document text는 DynamoDB에, embedding은 S3 Vectors에 저장하는 식으로 다른 AWS service와 조합할 수 있다.

기존 AWS account를 연결하면 Vercel platform에서 새로 만든 bucket만 integration 대상으로 사용할 수 있다. resource는 AWS account 안에 남으므로 CloudTrail, AWS Config, IAM과 cost management 도구를 계속 적용할 수 있다.

## 보안 및 운영 관점

- static access key 대신 OIDC federation과 STS temporary credential을 사용한다.
- Vercel에 연결되는 IAM role은 대상 bucket과 필요한 API action으로 권한을 제한한다.
- OIDC trust policy에서 허용할 Vercel project, environment와 account 조건을 검토한다.
- production과 preview environment는 서로 다른 role 또는 bucket으로 분리한다.
- S3 Block Public Access를 기본으로 유지하고 public delivery가 필요하면 CloudFront를 사용한다.
- bucket encryption, versioning, lifecycle와 access logging을 workload 특성에 맞게 설정한다.
- file upload에서는 content type, object size, key와 malware 검사 절차를 검증한다.
- S3 Tables는 schema evolution, snapshot retention과 compaction 상태를 관찰한다.
- S3 Vectors에는 원문 민감 데이터 대신 필요한 embedding과 최소 metadata만 저장한다.
- CloudTrail data event와 application log를 이용해 object 및 vector API 사용을 추적한다.
- Region 선택 시 user latency, data residency와 transfer 비용을 함께 고려한다.

## 비용 및 가용성

integration 자체에는 별도 요금이 없고 생성한 resource에 표준 Amazon S3 요금이 적용된다. storage, request, analytics query와 data transfer를 각각 고려해야 한다.

새 AWS account를 Vercel integration에서 만들면 6개월 동안 사용할 수 있는 AWS credit 100달러가 제공된다. 기존 account를 연결하면 현재 AWS billing을 그대로 사용한다.

Amazon S3, S3 Tables와 S3 Vectors integration은 Vercel이 지원하는 모든 commercial AWS Region에서 사용할 수 있다.

## 시험 관점 정리

- Amazon S3는 object storage이며 bucket과 object key를 사용해 데이터를 저장한다.
- S3 Tables는 Apache Iceberg table을 위한 managed storage와 자동 maintenance를 제공한다.
- S3 Vectors는 vector 저장과 similarity search를 제공하지만 embedding model은 제공하지 않는다.
- OIDC federation은 외부 workload가 static AWS key 없이 identity token으로 role을 assume하게 한다.
- AWS STS credential은 access key, secret key와 session token으로 구성되며 만료 시간이 있다.
- IAM role trust policy는 누가 role을 assume할 수 있는지, permission policy는 assume 후 무엇을 할 수 있는지 정한다.
- S3 Block Public Access는 의도하지 않은 public bucket 또는 object 노출을 막는 방어 계층이다.
- Apache Iceberg는 schema evolution, partition evolution와 snapshot 기반 transaction을 지원하는 open table format이다.
- RAG는 검색한 외부 지식을 model prompt에 추가해 답변의 근거와 최신성을 높이는 방식이다.

## 같이 보면 좋은 학습 포인트

- Amazon S3 general purpose bucket 보안 설정
- OIDC federation과 AWS STS AssumeRoleWithWebIdentity
- IAM trust policy와 least privilege
- S3 Tables 및 Apache Iceberg
- S3 Vectors와 vector database 비교
- Amazon Bedrock embedding model
- DuckDB에서 Iceberg table query하기
- S3 event와 CloudTrail data event
- multi-environment credential isolation
- Vercel Function과 AWS SDK v3

## 한 줄 정리

Amazon S3의 Vercel Marketplace 및 v0 integration은 object, Iceberg table과 vector storage를 Vercel에서 바로 만들고 OIDC와 STS의 단기 자격 증명으로 AWS account 안의 resource에 안전하게 연결한다.
