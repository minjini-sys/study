# AWS Lambda가 실행 중인 가용 영역을 알려주는 메타데이터 엔드포인트

- 정리 날짜: 2026-10-10
- 원문 게시일: 2026-10-09
- 공식 출처: [AWS Compute Blog - Reducing cross-AZ latency with the AWS Lambda metadata endpoint](https://aws.amazon.com/blogs/compute/reducing-cross-az-latency-with-the-aws-lambda-metadata-endpoint/)

## 핵심 내용

AWS Lambda가 실행 환경의 Availability Zone ID를 조회할 수 있는 메타데이터 엔드포인트를 제공한다. 기존에는 Lambda가 여러 가용 영역에 실행 환경을 자동 배치하더라도 함수 코드가 자신이 어느 AZ에서 실행되는지 직접 확인할 방법이 없었다.

함수는 실행 환경 내부의 loopback HTTP API에 인증된 GET 요청을 보내 `use1-az1` 같은 AZ ID를 얻을 수 있다. AWS는 각 실행 환경에 다음 환경 변수를 자동으로 제공한다.

- `AWS_LAMBDA_METADATA_API`: 메타데이터 서버의 주소와 포트
- `AWS_LAMBDA_METADATA_TOKEN`: 현재 실행 환경 전용 인증 토큰

Python, TypeScript, Java와 .NET에서는 Powertools for AWS Lambda의 metadata utility를 사용할 수 있다. custom runtime이나 container image에서는 엔드포인트를 직접 호출할 수도 있다.

이 기능은 Lambda를 지원하는 모든 commercial AWS Region에서 추가 비용 없이 사용할 수 있다. VPC 함수, custom runtime, container image, provisioned concurrency와 SnapStart도 지원한다.

## 왜 중요한가

Lambda가 ElastiCache, RDS/Aurora read replica, MemoryDB, EKS 또는 ECS처럼 AZ별 endpoint나 target을 가진 서비스에 접근할 때 다른 AZ의 자원을 선택하면 network latency와 cross-AZ data transfer 비용이 발생할 수 있다.

이제 함수가 자신의 AZ ID를 기준으로 같은 AZ의 endpoint를 우선 선택할 수 있다. latency-sensitive read path에서는 p99 지연과 데이터 전송 비용을 줄일 수 있고, 같은 AZ 자원이 장애 상태라면 다른 AZ로 fallback하여 고가용성도 유지할 수 있다.

AZ-aware routing은 성능 최적화뿐 아니라 특정 AZ 장애를 가정한 fault injection test, topology-aware routing과 multi-account 환경에서 물리적 AZ를 일관되게 식별하는 데도 활용할 수 있다.

## 메타데이터 엔드포인트 동작 방식

요청 경로는 다음과 같다.

```text
GET http://${AWS_LAMBDA_METADATA_API}/2026-01-15/metadata/execution-environment
Authorization: Bearer ${AWS_LAMBDA_METADATA_TOKEN}
```

응답은 실행 환경의 AZ ID를 포함한다.

```json
{
  "AvailabilityZoneID": "use1-az1"
}
```

- 엔드포인트는 Lambda 실행 환경 내부에서 runtime과 extension이 접근할 수 있다.
- 인증 토큰은 실행 환경마다 무작위로 생성되며 모든 요청에 Bearer token으로 전달해야 한다.
- GET 요청만 허용하며 token이 없거나 유효하지 않으면 `401 Unauthorized`가 반환된다.
- 응답은 실행 환경 안에서 변하지 않으므로 매 invocation마다 호출하지 않고 초기화 시 조회해 cache하는 것이 좋다.
- 응답의 기본 cache TTL은 43,200초이며 immutable로 표시된다.
- SnapStart는 restore 후 다른 AZ에서 실행될 수 있으므로 restore 이후 값을 갱신해야 한다. Powertools는 이 cache invalidation을 자동으로 처리한다.

## 서비스 및 아키텍처 관점

1. Lambda execution environment가 여러 AZ에 분산되어 함수의 가용성을 높인다.
2. cold start 또는 execution environment 초기화 과정에서 metadata endpoint를 호출해 AZ ID를 얻는다.
3. application configuration에는 AZ ID별 downstream endpoint mapping을 저장한다.
4. 함수는 같은 AZ의 read replica, cache node, pod 또는 task를 우선 선택한다.
5. 같은 AZ에 정상 endpoint가 없으면 다른 AZ의 healthy endpoint로 fallback한다.
6. write path는 데이터 일관성을 위해 primary endpoint로 보내고, read path만 AZ affinity를 적용할 수 있다.

Aurora에서는 읽기 요청을 함수와 같은 AZ의 read replica로 보낼 수 있다. MemoryDB에서는 eventual consistency를 허용하는 read를 same-AZ replica로 보내되, read-after-write가 필요한 요청은 primary로 보내야 한다.

EKS와 ECS에서는 Kubernetes Topology Aware Routing, 내부 load balancer 설정 또는 AWS Cloud Map의 service discovery와 AZ ID를 결합할 수 있다. ElastiCache처럼 AZ별 node endpoint가 있는 경우에는 configuration에서 AZ ID와 endpoint를 mapping해 가장 가까운 node를 선택한다.

## 보안 및 운영 관점

- metadata token을 log, error message나 외부 응답에 노출하지 않는다.
- token을 환경 변수에서 읽고 요청마다 `Authorization` header에 전달한다.
- endpoint 주소를 hardcode하지 않고 `AWS_LAMBDA_METADATA_API`를 사용한다.
- 각 실행 환경에서 한 번 조회해 cache하고 매 invocation의 불필요한 HTTP 호출을 피한다.
- same-AZ endpoint가 없거나 장애 상태일 때 사용할 cross-AZ fallback을 반드시 둔다.
- AZ별 latency, error rate, connection count와 data transfer 비용을 CloudWatch에서 함께 관찰한다.
- AZ affinity 때문에 특정 replica나 cache node에 부하가 집중되지 않는지 확인한다.
- AZ name인 `us-east-1a`와 AZ ID인 `use1-az1`을 혼동하지 않는다. AZ ID는 계정이 달라도 동일한 물리적 AZ를 가리킨다.
- AZ ID를 AZ name으로 변환해야 하면 EC2 `DescribeAvailabilityZones` API와 최소 권한인 `ec2:DescribeAvailabilityZones`를 사용한다.
- AZ fault injection test에서는 의도한 AZ의 traffic과 dependency만 격리되는지 검증한다.

## 시험 관점 정리

- Lambda는 고가용성을 위해 한 Region의 여러 Availability Zone에 execution environment를 배치한다.
- Availability Zone name은 계정별로 다른 물리 AZ에 mapping될 수 있지만 AZ ID는 계정 간에 일관된다.
- same-AZ routing은 latency와 cross-AZ data transfer 비용을 줄이지만 multi-AZ fallback을 제거하면 가용성이 낮아질 수 있다.
- Powertools for AWS Lambda는 metadata 조회, cache와 SnapStart restore 이후 갱신을 단순화한다.
- provisioned concurrency는 실행 환경을 미리 준비하고 SnapStart는 초기화된 snapshot을 restore해 cold start를 줄인다.
- read replica와 cache replica는 읽기 성능을 높일 수 있지만 consistency requirement에 따라 primary 사용이 필요하다.
- topology-aware routing은 요청을 가까운 target으로 보내지만 health check와 fallback policy를 함께 설계해야 한다.

## 같이 보면 좋은 학습 포인트

- AWS Region, Availability Zone name과 AZ ID의 차이
- Lambda execution environment lifecycle과 cold start
- Lambda SnapStart와 provisioned concurrency 비교
- Powertools for AWS Lambda metadata utility
- Aurora reader endpoint와 read replica load balancing
- ElastiCache 및 MemoryDB의 primary/replica 구조
- Kubernetes Topology Aware Routing
- cross-AZ data transfer 비용 구조
- AWS Fault Injection Service를 이용한 AZ 장애 실험

## 한 줄 정리

Lambda metadata endpoint는 함수의 AZ ID를 안전하게 제공해 같은 AZ의 downstream resource를 우선 사용하고 latency와 cross-AZ 비용을 줄이면서 fallback 기반 고가용성을 유지하게 한다.
