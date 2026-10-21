# Amazon ECS Gateway: ECS 서비스용 완전 관리형 ingress

- 정리 날짜: 2026-10-21
- 원문 게시일: 2026-10-20
- 공식 출처: [AWS News Blog - Introducing Amazon ECS Gateway: A fully-managed ingress solution for ECS services](https://aws.amazon.com/blogs/aws/introducing-amazon-ecs-gateway-a-fully-managed-ingress-solution-for-ecs-services/)

## 핵심 내용

AWS가 Amazon ECS service의 ingress networking을 자동으로 구성하고 관리하는 새로운 account-level resource인 **Amazon ECS Gateway**를 출시했다.

기존에는 ECS service를 외부 또는 VPC 내부에 공개할 때 application team이나 platform team이 Application Load Balancer(ALB), target group, listener rule, AWS Certificate Manager(ACM) certificate, security group, DNS와 AWS WAF web ACL을 직접 연결하고 lifecycle을 관리해야 했다.

ECS Gateway에서는 platform team이 subnet, security group, TLS certificate, listener, WAF와 public 또는 internal scheme 같은 공통 network policy를 gateway에 정의한다. application team은 ECS service 생성 또는 수정 시 gateway connector만 추가한다. 그러면 ECS가 service별 DNS hostname, ACM certificate, listener rule, target group과 service security group을 자동으로 만든다.

하나의 ALB는 host-based routing을 사용해 최대 25개 service가 공유할 수 있다. ECS는 연결된 service 수와 ALB capacity에 따라 ALB를 추가하거나 제거하고, service가 삭제되면 관련 route와 resource도 정리한다.

## 왜 중요한가

container platform에서는 application 배포보다 ingress resource 구성과 team 간 승인 절차가 더 오래 걸리는 경우가 있다. service마다 ALB와 network 설정을 따로 관리하면 비용이 늘고 TLS, WAF와 security group 정책도 서로 달라질 수 있다.

ECS Gateway는 platform team이 조직 표준을 한 곳에 정의하고 application team이 self-service 방식으로 이를 사용하게 한다. application developer는 load balancer의 세부 설정을 반복하지 않고 service port와 access type 같은 application 요구 사항에 집중할 수 있다.

또한 실제 ALB와 관련 resource는 AWS가 소유한 별도 account가 아니라 사용자의 account 안에 생성된다. 기존 CloudWatch monitoring, IAM governance, tagging과 cost management 흐름을 계속 사용할 수 있다.

## 동작 방식

### 1. Platform team이 gateway 생성

gateway에는 다음 항목을 설정한다.

- ECS가 network resource를 생성하고 관리할 때 assume할 infrastructure IAM role
- internet-facing 또는 internal load balancing scheme
- VPC와 subnet
- IPv4 등 IP address type
- listener protocol과 port
- gateway가 확장할 수 있는 최대 ALB 수
- TLS certificate와 security group
- 선택적인 AWS WAF web ACL

gateway 생성만으로는 ALB가 바로 만들어지지 않는다. service가 gateway connector를 통해 연결될 때 실제 infrastructure가 provision된다.

일부 설정은 gateway 생성 후 변경할 수 없다. 이름, scheme, VPC, subnet, IP address type, listener protocol과 port를 미리 정확히 설계해야 한다.

### 2. Gateway와 ECS cluster 연결

하나의 gateway는 최대 10,000개 cluster와 연결할 수 있고 하나의 cluster는 최대 5개 gateway association을 가질 수 있다. 예를 들어 같은 cluster에 public gateway와 private gateway를 각각 연결할 수 있다.

association은 어떤 cluster의 service가 어떤 gateway를 사용할 수 있는지 정하는 platform boundary 역할을 한다. IAM policy로 특정 cluster에 service를 배포할 수 있는 principal도 제한할 수 있다.

### 3. Application team이 gateway connector 추가

application team은 `CreateService` 또는 `UpdateService`에서 gateway connector를 지정한다. connector에는 service의 named port, public 또는 private access filter와 health check 같은 정보를 넣는다.

deployment 시 ECS는 cluster에 연결된 gateway 가운데 다음 조건을 만족하는 gateway를 찾는다.

- connector가 요청한 access type과 gateway가 일치한다.
- gateway와 ECS service가 같은 VPC에 있다.
- gateway가 service subnet이 사용하는 모든 Availability Zone을 포함한다.

조건이 맞으면 ECS는 capacity가 남은 ALB에 service를 배치하고 hostname 기반 listener rule을 만든다. ALB capacity가 없으면 `maxAlbCount` 범위 안에서 새 ALB를 생성한다.

### 4. Lifecycle 자동 관리

- service 재배포 시 현재 cluster association을 다시 평가한다.
- 새 gateway가 연결되면 compatible한 service가 다음 deployment에서 연결될 수 있다.
- gateway association이 제거되면 route도 제거된다.
- service 삭제 시 target group, listener rule, certificate와 service security group을 정리한다.
- 마지막 route가 제거된 ALB는 자동으로 deprovision된다.

## 서비스 및 아키텍처 관점

전형적인 요청 흐름은 다음과 같다.

1. client가 service별 unique DNS hostname으로 HTTPS request를 보낸다.
2. public 또는 internal ALB가 request를 수신한다.
3. ALB listener가 host header를 기준으로 해당 service의 rule을 선택한다.
4. rule이 ECS service 전용 target group으로 traffic을 전달한다.
5. target group이 `awsvpc` network mode의 ECS task ENI로 request를 전달한다.
6. service security group은 gateway ALB security group에서 오는 inbound traffic만 허용한다.
7. WAF web ACL을 연결한 경우 ALB 앞에서 application-layer request를 검사한다.
8. CloudWatch ALB metric과 target health로 service 상태를 관찰한다.

여러 service가 하나의 ALB를 공유하므로 ALB 비용을 줄일 수 있지만 blast radius와 quota를 함께 고려해야 한다. gateway 하나의 설정 변경이나 shared ALB 문제가 여러 service에 영향을 줄 수 있으므로 workload 중요도와 environment에 따라 gateway를 분리하는 전략이 필요하다.

기존 ECS service에 이미 load balancer가 있어도 `UpdateService`로 gateway connector를 추가해 점진적으로 이전할 수 있다. 검증 기간에는 기존 ingress와 ECS Gateway ingress를 함께 유지한 뒤 traffic test가 끝나면 기존 구성을 제거할 수 있다.

## 보안 관점

- ECS infrastructure role에는 ALB, ACM, security group과 관련 resource를 관리하는 데 필요한 최소 권한만 부여한다.
- application team과 platform team의 역할을 분리하고 gateway 생성 및 cluster association 권한을 제한한다.
- internet-facing gateway와 internal gateway를 분리해 의도하지 않은 public exposure를 방지한다.
- service security group이 ALB security group에서 오는 traffic만 허용하는지 확인한다.
- TLS certificate lifecycle과 hostname 소유권을 점검하고 HTTPS listener를 기본값으로 사용한다.
- AWS WAF managed rule과 application별 exception을 중앙 정책에 맞게 구성한다.
- shared ALB를 사용하는 service 사이에서 listener rule, hostname과 tag가 올바르게 분리되는지 감사한다.
- CloudTrail로 gateway 생성, association과 ECS service update action을 기록한다.
- gateway와 service가 사용하는 subnet 및 Availability Zone 범위를 검토해 우회 경로나 잘못된 network exposure가 생기지 않게 한다.

## 운영 관점

ECS Gateway는 `awsvpc` network mode를 사용하는 ECS service에서 지원되며 ECS on EC2, AWS Fargate와 ECS Managed Instances에 적용할 수 있다.

service가 사용하는 Availability Zone은 gateway가 구성된 Availability Zone의 부분 집합이어야 한다. gateway ALB는 subnet이 있는 AZ의 task에만 traffic을 전달하므로 같은 cluster의 gateway와 service는 가능한 한 동일한 AZ 집합으로 구성하는 것이 좋다.

기본 `maxAlbCount` quota는 10이다. 최대 ALB 수에 도달하고 기존 ALB에도 capacity가 없으면 deployment가 실패하므로 예상 service 수, service당 route 수와 성장률을 기준으로 capacity를 계획해야 한다.

monitoring에서는 다음 항목을 구분한다.

- `TargetGroup` dimension: service별 traffic과 target 상태
- `LoadBalancer` dimension: shared ALB 전체 traffic과 error
- `HealthyHostCount`: 정상 task 수
- `TargetResponseTime`: backend 응답 시간
- `HTTPCode_Target_5XX_Count`: target이 반환한 5xx error
- `describe-target-health`: 개별 target의 실시간 health 상태

gateway ARN 기반 tag를 사용해 gateway가 ALB를 추가해도 dashboard와 alarm이 자동으로 확장되도록 구성하는 것이 좋다.

ECS Gateway 자체에는 추가 요금이 없지만 사용자의 account에 생성되는 ALB, CloudWatch log와 metric, ACM certificate 종류, data transfer 등 underlying infrastructure 비용은 발생한다.

## 도입 전 확인할 질문

1. public과 internal ingress를 하나의 gateway로 운영할지 분리할지 결정했는가?
2. gateway를 environment, team 또는 workload criticality 중 어떤 경계로 나눌 것인가?
3. 예상 service 수에 맞춰 `maxAlbCount`와 관련 quota를 계산했는가?
4. gateway와 service의 VPC 및 Availability Zone 구성이 일치하는가?
5. infrastructure role이 최소 권한으로 구성되어 있는가?
6. shared ALB 장애와 WAF 정책 변경의 blast radius를 평가했는가?
7. 기존 ingress에서 점진적으로 이전하고 rollback할 절차가 있는가?
8. service별 metric, log, target health와 cost를 구분할 tagging이 준비되어 있는가?

## 시험 관점 정리

- Amazon ECS는 containerized application을 실행하고 관리하는 orchestration service다.
- ALB는 application layer에서 host와 path 기반 routing을 지원한다.
- target group은 load balancer가 traffic을 전달할 destination 집합이다.
- `awsvpc` network mode에서는 각 ECS task가 VPC의 ENI와 private IP를 가진다.
- internet-facing ALB는 public client traffic을 받고 internal ALB는 VPC 내부에서 사용한다.
- security group은 stateful virtual firewall이다.
- AWS Certificate Manager는 TLS certificate의 provisioning과 renewal을 관리한다.
- AWS WAF는 web request를 규칙에 따라 허용, 차단 또는 관찰한다.
- IAM role의 trust policy는 누가 role을 assume할 수 있는지 정의한다.
- host-based routing은 HTTP `Host` header에 따라 서로 다른 target group으로 전달한다.
- shared infrastructure는 비용을 줄일 수 있지만 quota와 blast radius 관리가 필요하다.
- CloudWatch dimension은 metric을 resource 또는 구성 요소별로 구분하는 key-value pair다.

## 같이 보면 좋은 학습 포인트

- Amazon ECS service와 task definition
- ECS `awsvpc` network mode
- Application Load Balancer listener와 rule
- host-based routing과 path-based routing
- target group health check
- public subnet과 private subnet
- ACM TLS certificate lifecycle
- AWS WAF web ACL
- IAM infrastructure role과 least privilege
- CloudWatch ALB metric과 alarm
- ECS deployment circuit breaker
- shared ingress의 quota와 blast radius

## 한 줄 정리

Amazon ECS Gateway는 platform team이 account 수준의 ingress 표준을 정의하면 ECS가 service별 ALB route, DNS, TLS certificate, target group과 security group lifecycle을 자동 관리해 container 배포 속도와 network configuration 일관성을 높이는 기능이다.
