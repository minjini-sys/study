# AWS Resilience Hub의 Dependency Insights와 복원력 정책 공유

- 정리 날짜: 2026-10-20
- 원문 게시일: 2026-10-13
- 공식 출처: [AWS Architecture Blog - Scope workloads and share resilience policies with AWS Resilience Hub](https://aws.amazon.com/blogs/architecture/scope-workloads-and-share-resilience-policies-with-aws-resilience-hub/)

## 핵심 내용

AWS가 차세대 AWS Resilience Hub에서 application의 실제 resource와 복원력 요구 사항을 더 정확하게 연결하는 세 가지 기능을 소개했다.

1. Amazon EKS label selector를 사용해 shared cluster와 namespace 안에서 특정 service에 속한 Kubernetes resource만 선택할 수 있다.
2. 생성형 AI 기반 Dependency Insights가 DNS activity에서 발견한 dependency를 분석해 cross-Region, 신규, third-party와 uneven usage pattern을 우선순위별로 보여준다.
3. 중앙 platform 또는 SRE team이 AWS Organizations를 통해 resilience policy를 member account에 공유할 수 있다.

application은 배포와 migration을 반복하면서 dependency가 계속 바뀐다. 문서의 architecture와 실제 runtime dependency가 달라지면 RTO와 RPO가 올바르게 정의되어 있어도 장애 복구 계획은 실패할 수 있다. 이번 기능은 resource scope, 실제 dependency와 중앙 복원력 기준을 같은 흐름에서 검토하도록 돕는다.

## 왜 중요한가

복원력 평가는 어떤 resource가 하나의 service를 구성하는지 정확히 알아야 시작할 수 있다. 여러 team이 하나의 EKS cluster와 namespace를 공유하면 namespace 단위 선택만으로는 서로 다른 workload가 한 assessment에 섞일 수 있다. Kubernetes label을 그대로 활용하면 cluster를 재구성하지 않고도 service boundary를 좁힐 수 있다.

또한 architecture diagram에는 database와 queue 같은 주요 component만 보이지만 실제 workload는 다른 Region의 AWS endpoint, SaaS API, monitoring endpoint와 container registry에도 의존한다. 이 dependency가 장애 시 hard dependency인지 soft dependency인지 확인하지 않으면 service 자체가 정상이어도 전체 user journey가 중단될 수 있다.

마지막으로 각 account가 RTO와 RPO를 따로 작성하면 같은 중요도의 workload에 서로 다른 기준이 적용될 수 있다. Organizations 기반 policy 공유는 중앙 team이 공통 기준을 정의하고 application owner가 일관된 target을 사용하도록 한다.

## 세 가지 주요 기능

### 1. EKS label 기반 service 범위 지정

Resilience Hub의 service input source에는 CloudFormation stack, Terraform state, resource tag와 EKS cluster를 사용할 수 있다. EKS input source는 cluster와 namespace를 지정한 뒤 Kubernetes label selector로 resource를 추가로 걸러낼 수 있다.

예를 들어 `production` namespace에 checkout과 inventory가 함께 있어도 `app In checkout` selector를 사용하면 checkout의 Deployment와 ReplicaSet만 service 범위에 포함할 수 있다. 한 namespace의 여러 selector는 `AND` 방식으로 평가된다.

단, label은 Kubernetes object의 범위를 좁힐 뿐 VPC, subnet, cluster와 node group 같은 shared AWS infrastructure를 checkout 전용 resource로 바꾸지는 않는다. 평가 결과에서 shared resource와 owner를 따로 확인해야 한다.

### 2. 생성형 AI 기반 Dependency Insights

Dependency Insights는 Resilience Hub의 dependency discovery 결과를 요약하고 확인할 가치가 높은 pattern을 분류한다.

- Cross-Region dependency: 다른 Region 호출이 의도된 것인지, Region 복구 설계와 맞는지 확인한다.
- New dependency: 최근 7일 안에 처음 관찰된 endpoint가 architecture 변경을 의미하는지 확인한다.
- Third-party dependency: 외부 provider가 중단될 때 service가 어떤 상태로 동작하는지 확인한다.
- Uneven usage: 특정 시간에만 나타나는 dependency가 batch job인지 deployment 과정의 호출인지 조사한다.

dependency discovery는 Amazon Route 53 Resolver를 통한 DNS activity를 분석하며 workload에 agent를 설치하지 않는다. 최대 35일의 사용 가능한 history를 살펴보고 이후의 변화도 계속 관찰한다. endpoint가 수백 개일 때 Insights가 조사 순서를 좁혀주지만 자동으로 해결책을 결정하지는 않는다.

Insights는 24시간에 한 번 다시 생성할 수 있다. 신규 service는 충분한 history가 쌓이기 전에는 결과가 제한적일 수 있다.

### 3. AWS Organizations 기반 policy 공유

management account 또는 delegated administrator가 resilience policy를 만들고 organization의 member account에 공유할 수 있다. policy는 다음 네 가지 target을 조합한다.

- service availability target
- Availability Zone disruption에 대한 recovery target
- Regional disruption에 대한 recovery target
- data recovery target

RTO는 목표 복구 시간을 의미하고 RPO는 허용 가능한 데이터 손실 시간을 의미한다. 중앙 team은 policy attachment와 detachment event를 확인해 각 service의 채택 상태를 추적할 수 있다.

shared policy를 service에 연결했다는 사실은 policy 채택을 의미할 뿐 target을 충족했다는 증거는 아니다. failure mode assessment와 실제 resilience test를 수행해야 compliance를 판단할 수 있다.

## 서비스 및 아키텍처 관점

권장되는 assessment 흐름은 다음과 같다.

1. EKS label, CloudFormation stack, Terraform state 또는 resource tag로 service input source를 정의한다.
2. Resilience Hub가 service resource와 supporting infrastructure를 discovery한다.
3. 선택적으로 dependency discovery를 켜 Route 53 Resolver의 DNS activity를 분석한다.
4. Dependency Insights가 cross-Region, 신규, third-party와 uneven usage pattern을 요약한다.
5. service owner가 각 dependency를 hard 또는 soft로 분류하고 실제 architecture와 비교한다.
6. business owner와 SRE team이 availability, Multi-AZ, Multi-Region과 data recovery target을 합의한다.
7. 중앙 account가 Organizations를 통해 resilience policy를 공유한다.
8. workload account가 policy를 service에 연결하고 failure mode assessment를 수행한다.
9. AWS Fault Injection Service 같은 도구로 가정을 시험하고 RTO와 RPO 달성 여부를 검증한다.

hard dependency는 사용할 수 없으면 service가 실패하는 dependency이고, soft dependency는 일부 기능이나 성능이 저하되어도 핵심 service가 계속 동작하는 dependency다. 이 구분은 DNS volume만으로 자동 결정할 수 없으며 application owner의 검증과 failure test가 필요하다.

## 보안 관점

- Resilience Hub가 resource를 읽을 수 있도록 전용 IAM role을 사용하고 필요한 discovery 및 assessment permission만 부여한다.
- EKS resource를 조사하는 role에는 EKS access entry와 Kubernetes RBAC를 함께 설정해야 한다.
- 중앙 policy 관리 권한과 workload assessment 권한을 분리해 한 account 또는 role에 과도한 권한이 모이지 않게 한다.
- Organizations trusted access와 service-linked role 생성 여부를 변경 관리 대상으로 기록한다.
- dependency 결과에는 내부 endpoint와 외부 service 이름이 포함될 수 있으므로 열람 권한과 log 보존 정책을 관리한다.
- third-party endpoint가 발견되면 인증 정보, data flow, network egress와 공급자 장애 시 fallback을 검토한다.
- label이 누락되거나 잘못 적용되면 assessment scope에서 resource가 빠질 수 있으므로 admission policy와 CI validation을 고려한다.
- 생성형 AI Insight는 조사 지원 정보이며 보안 또는 architecture 변경을 자동 승인하는 근거로 사용하지 않는다.

## 운영 관점

Dependency Insights를 사용하기 전에 service owner와 architecture가 명확한 workload부터 시작하는 것이 좋다. 발견된 endpoint가 정상 traffic인지 설명할 수 있어야 Insight의 정확성과 누락을 판단할 수 있다.

운영 시에는 다음 사항을 확인한다.

- IP address로 직접 연결한 dependency는 DNS 기반 discovery에 나타나지 않는다.
- Lambda function은 VPC connectivity가 있어야 해당 방식의 dependency discovery 범위에 들어간다.
- 호출 빈도가 매우 낮은 dependency는 초기 결과에서 보이지 않을 수 있다.
- container image pull처럼 deployment 때만 발생하는 traffic은 uneven usage로 표시될 수 있다.
- cross-Region 호출은 자동으로 잘못된 architecture라는 뜻이 아니며 의도와 복구 영향도를 검증해야 한다.
- shared policy의 attachment event는 준수 증거가 아니라 채택 기록이다.
- policy 공유를 끄면 이를 사용하던 member service에서 policy가 제거되므로 team 간 변경 조율이 필요하다.
- dependency discovery는 선택 기능이며 별도 비용이 발생하므로 대상 service와 관찰 기간을 계획한다.

복원력 평가는 일회성 보고서가 아니라 deployment 이후 반복되는 운영 절차여야 한다. dependency change, policy attachment 변경과 assessment 결과를 deployment 및 incident review workflow에 연결하면 문서와 실제 환경의 차이를 줄일 수 있다.

## 시험 관점 정리

- AWS Resilience Hub는 application의 resilience posture를 정의하고 평가하며 개선하는 managed service다.
- RTO는 장애 후 service를 복구해야 하는 목표 시간이다.
- RPO는 장애 시 허용할 수 있는 데이터 손실 시간 범위다.
- Multi-AZ는 하나의 Region 안에서 Availability Zone 장애에 대비하는 방식이다.
- Multi-Region은 Regional disruption까지 고려하는 architecture다.
- AWS Organizations trusted access는 AWS service가 organization의 account와 연동하도록 허용한다.
- service-linked role은 특정 AWS service가 사용하도록 사전 정의된 IAM role이다.
- Kubernetes label selector는 label key, operator와 value로 object를 선택한다.
- RBAC는 Kubernetes resource에 대한 role 기반 access control이다.
- Amazon Route 53 Resolver는 VPC의 DNS query를 처리하며 dependency discovery의 관찰 지점으로 사용된다.
- hard dependency 장애는 핵심 service 실패로 이어지고 soft dependency 장애는 degraded mode로 처리할 수 있다.
- resilience policy를 연결하는 것과 실제 RTO 및 RPO를 만족하는 것은 서로 다르다.
- AWS Fault Injection Service는 controlled failure를 주입해 resilience 가정을 시험하는 서비스다.

## 같이 보면 좋은 학습 포인트

- AWS Resilience Hub의 service, system과 user journey 계층
- RTO와 RPO 설계 방법
- Multi-AZ와 Multi-Region disaster recovery pattern
- EKS namespace, label selector와 RBAC
- Route 53 Resolver와 DNS query 흐름
- hard dependency와 soft dependency 분류
- AWS Organizations trusted access
- IAM service-linked role
- failure mode analysis
- AWS Fault Injection Service 실험 설계
- third-party dependency와 circuit breaker
- recovery runbook과 game day 운영

## 한 줄 정리

AWS Resilience Hub는 EKS label로 service 범위를 정확히 정하고 Dependency Insights로 실제 runtime dependency를 우선순위화하며 Organizations 공유 policy로 RTO와 RPO 기준을 중앙화하지만, 최종 복원력은 application owner의 검증과 실제 장애 시험으로 확인해야 한다.
