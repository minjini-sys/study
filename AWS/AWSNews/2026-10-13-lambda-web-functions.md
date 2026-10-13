# 표준 HTTP 애플리케이션을 실행하는 AWS Lambda Web Functions 출시

- 정리 날짜: 2026-10-13
- 원문 게시일: 2026-10-12
- 공식 출처: [AWS News Blog - Announcing AWS Lambda Web Functions](https://aws.amazon.com/blogs/aws/announcing-aws-lambda-web-functions-for-fast-consistent-web-app-performance-anywhere/)

## 핵심 내용

AWS가 표준 HTTP web application을 event handler로 다시 작성하지 않고 Lambda에서 실행할 수 있는 AWS Lambda Web Functions를 출시했다.

Express, Hono, Fastify, Next.js와 같은 Node.js framework로 만든 port-listening server를 그대로 배포할 수 있다. Lambda-specific adapter나 wrapper 없이 application이 `AWS_LAMBDA_HTTP_ENDPOINT` 환경 변수에 지정된 host와 port에서 listen하면 된다.

Web Functions는 하나의 execution environment에서 최대 128개의 요청을 동시에 처리한다. CPU를 실제로 사용한 시간에 대해 active CPU billing을 적용하고, memory는 execution environment가 존재하는 동안 과금한다. database query, external API 또는 token streaming을 기다리는 시간이 많은 I/O-heavy web workload의 compute 비용을 줄일 수 있다.

## Lambda의 세 가지 form factor

### Event Functions

기존 Lambda function에 붙은 이름이다. event를 받아 하나의 concurrent handler가 처리하고 응답하며 wall-clock time을 기준으로 과금한다. queue consumer, event processing와 다른 AWS service가 trigger하는 비동기 workload에 적합하다.

### Web Functions

표준 HTTP server가 port에서 요청을 받고 하나의 execution environment가 여러 요청을 동시에 처리한다. API backend, server-side rendering와 interactive HTTP workload에 적합하다.

### Lambda MicroVMs

customer-managed lifecycle을 가진 isolated, stateful sandbox다. untrusted 또는 user-generated code를 실행하는 용도에 적합하다.

세 form factor는 모두 Firecracker isolation 기술을 사용한다. 하나의 application에서 비동기 처리는 Event Functions, user-facing API와 SSR은 Web Functions, interactive code execution은 MicroVMs로 나눠 사용할 수 있다.

## 왜 중요한가

기존 HTTP application을 Lambda에 옮길 때는 Lambda event 형식에 맞춘 adapter, framework integration과 request translation이 필요했다. global service를 만들려면 여러 Region에 별도로 배포하고 latency-based routing도 구성해야 했다.

Web Functions는 일반 Node.js server programming model을 유지하면서 serverless scaling과 managed HTTPS endpoint를 제공한다. 개발자는 framework를 바꾸지 않고 기존 web code를 배포할 수 있으며, 여러 요청이 warm environment를 공유하므로 요청마다 execution environment가 필요하지 않다.

active CPU billing은 network와 database I/O 대기가 많은 workload에 유리하다. 다만 memory는 environment lifecycle 전체에 대해 과금되므로 concurrency, memory size와 workload 특성을 함께 측정해야 한다.

## 배포 및 revision 관리

다음 명령은 source를 package하고 Web Function을 생성한 뒤 첫 immutable revision을 publish하고 HTTPS endpoint를 연결한다.

```bash
aws lambda-web deploy --name my-web-app --code ./my-app
```

revision은 배포된 code의 immutable snapshot이다. endpoint는 하나의 revision으로 traffic을 보내거나 여러 revision 사이에 traffic을 분할한다. 새 version을 배포한 뒤 endpoint weight를 조정하면 별도 deployment 도구 없이 canary release를 구성할 수 있다.

예를 들어 기존 revision에 90%, 새 revision에 10%를 전달해 production traffic으로 새 code를 검증할 수 있다. 문제가 확인되면 traffic weight를 이전 revision으로 되돌리는 방식으로 빠르게 rollback할 수 있다.

## 멀티 리전 동작

배포 명령에 여러 Region을 지정하면 Web Functions가 code를 각 Region으로 복제하고 latency-based routing을 자동으로 구성한다.

```bash
aws lambda-web deploy --name my-web-app --code ./my-app \
  --regions us-east-1,eu-west-1,ap-northeast-1
```

사용자는 하나의 global URL로 접속하고 요청은 best-effort 방식으로 호출자에게 latency가 낮은 Region으로 전달된다. application data도 여러 Region에서 필요하다면 database replication, consistency, failover와 data residency는 별도로 설계해야 한다.

## CloudFront 연동

Web Function endpoint를 Amazon CloudFront distribution의 origin으로 사용할 수 있다. static asset은 edge cache에서 제공하고 dynamic request는 Web Function으로 전달하는 구조를 만들 수 있다.

CloudFront Origin Access Control(OAC)을 Web Function endpoint에 연결하면 CloudFront가 origin request에 서명하고 endpoint는 지정한 distribution의 호출만 허용한다. public endpoint를 직접 노출하거나 application에 별도 origin authorization을 구현하지 않고 private origin pattern을 적용할 수 있다.

## 서비스 및 아키텍처 관점

1. 사용자가 CloudFront 또는 Web Function의 global URL로 HTTP 요청을 보낸다.
2. global endpoint는 latency를 기준으로 적합한 Region을 선택한다.
3. regional endpoint가 선택된 immutable revision으로 traffic을 전달한다.
4. Lambda Web Function execution environment가 표준 Node.js HTTP server를 실행한다.
5. 하나의 environment가 최대 128개 concurrent request를 처리한다.
6. VPC connectivity를 통해 private subnet의 database, cache와 internal service에 접근할 수 있다.
7. CloudWatch log와 metric으로 request, error, latency와 resource 사용량을 관찰한다.

출시 시점에는 Node.js 24 runtime과 Arm64 architecture를 사용한다. 지원 Region은 US East(N. Virginia), US East(Ohio), US West(Oregon), Asia Pacific(Tokyo)와 Europe(Ireland)다.

## 보안 및 운영 관점

- CloudFront OAC를 사용해 origin endpoint에 직접 접근하는 경로를 제한한다.
- public API에는 application authentication과 authorization을 별도로 구현한다.
- VPC 연결 시 private resource에 필요한 security group과 network path만 허용한다.
- execution role은 database secret, S3 object와 다른 AWS API에 필요한 최소 권한만 부여한다.
- credential은 code나 environment variable의 평문 값 대신 AWS Secrets Manager 또는 Parameter Store에서 관리한다.
- revision별 traffic weight와 error rate를 관찰하고 canary rollback 기준을 정한다.
- multi-concurrency 환경에서 global variable, connection pool과 shared mutable state가 thread-safe 또는 request-safe한지 확인한다.
- 한 environment의 128개 동시 요청이 downstream database connection을 고갈시키지 않도록 connection limit과 backpressure를 구성한다.
- multi-Region 배포에서는 data residency, session state, cache consistency와 database failover를 함께 검토한다.
- CloudWatch alarm, distributed tracing와 structured logging으로 regional 및 revision별 문제를 구분한다.
- memory billing과 active CPU 사용량을 load test해 적절한 memory size 및 concurrency를 선택한다.

## 시험 관점 정리

- Lambda Event Functions는 event-driven single-concurrent handler model을 사용한다.
- Lambda Web Functions는 HTTP-native multi-concurrent execution model을 사용한다.
- Lambda MicroVMs는 stateful sandbox와 customer-managed lifecycle을 제공한다.
- 세 form factor는 모두 Firecracker 기반 isolation을 사용한다.
- immutable revision은 배포 단위를 고정하고 endpoint weight로 traffic splitting을 가능하게 한다.
- canary deployment는 새 version에 일부 traffic만 보내 위험을 줄이는 방식이다.
- latency-based routing은 사용자를 지연 시간이 낮은 Region으로 연결하지만 application data의 multi-Region 복제를 자동으로 해결하지는 않는다.
- CloudFront는 content delivery network이며 OAC는 distribution에서 origin으로 가는 request를 보호한다.
- VPC connectivity는 Lambda가 private AWS resource에 접근하도록 하지만 subnet, route와 security group 설계가 필요하다.
- active CPU billing과 wall-clock billing은 I/O wait가 많은 workload에서 비용 특성이 달라진다.

## 같이 보면 좋은 학습 포인트

- Lambda Event Functions, Web Functions와 MicroVMs 비교
- Firecracker microVM isolation
- Node.js event loop와 multi-concurrent HTTP 처리
- CloudFront Origin Access Control
- immutable deployment와 canary release
- Route 53 latency-based routing과 global endpoint
- Lambda VPC networking 및 database connection 관리
- server-side rendering과 streaming response
- multi-Region data consistency 및 disaster recovery
- CloudWatch metric, log와 distributed tracing

## 한 줄 정리

AWS Lambda Web Functions는 표준 Node.js HTTP server를 수정 없이 배포하고 multi-concurrent execution, active CPU billing, global routing과 revision 기반 traffic shifting을 사용할 수 있게 하는 새로운 Lambda form factor다.
