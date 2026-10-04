# GuardDuty Runtime Monitoring, Security Hub Threat Analytics 플랜에 포함

- 정리 날짜: 2026-10-04
- 원문 게시일: 2026-10-02
- 공식 출처: [AWS What's New - GuardDuty Runtime Monitoring is now included in the AWS Security Hub Threat Analytics plan](https://aws.amazon.com/about-aws/whats-new/2026/10/aws-security-hub-runtime-monitoring/)

## 핵심 내용

Amazon GuardDuty Runtime Monitoring이 AWS Security Hub Threat Analytics 플랜에 포함되었다. Runtime Monitoring은 운영체제, 네트워크와 파일 활동을 관찰해 컨테이너 탈출, 권한 상승과 암호화폐 채굴 같은 런타임 위협을 탐지한다.

보호 대상에는 Amazon EC2 인스턴스, Amazon EKS 클러스터와 AWS Fargate에서 실행되는 Amazon ECS 태스크가 포함된다. 탐지 범위, finding 유형과 기존 GuardDuty 보안 에이전트는 그대로 유지되며 사용자가 별도로 다시 구성할 필요는 없다.

Security Hub가 계정과 리전에서 활성화되어 있다면 해당 범위의 Runtime Monitoring 사용량은 더 이상 GuardDuty에서 별도로 청구되지 않는다. EC2, EKS와 ECS on Fargate 사용량이 AWS Security Hub의 단일 사용 유형으로 측정되고 청구된다.

## 왜 중요한가

클라우드 보안에서는 설정 오류나 취약점뿐 아니라 실행 중인 워크로드 내부에서 발생하는 비정상 행동을 탐지해야 한다. 컨테이너가 격리 경계를 벗어나거나 프로세스가 권한을 높이고, 공격자가 컴퓨팅 자원을 채굴에 사용하는 행위는 control plane 로그만으로 놓칠 수 있다.

이번 변경은 GuardDuty의 런타임 탐지 기능을 Security Hub Threat Analytics 플랜의 위협 분석 흐름과 비용 체계에 통합한다. 여러 컴퓨팅 유형의 Runtime Monitoring 사용량을 Security Hub에서 하나의 사용 유형으로 확인할 수 있어 보안 서비스 비용과 적용 범위를 관리하기 쉬워진다.

## 탐지 대상과 역할

- **Amazon EC2**: 인스턴스 운영체제 수준의 프로세스, 네트워크와 파일 활동을 감시한다.
- **Amazon EKS**: Kubernetes 워크로드에서 발생하는 런타임 위협을 탐지한다.
- **Amazon ECS on AWS Fargate**: 서버를 직접 관리하지 않는 컨테이너 태스크의 런타임 활동을 감시한다.
- **Amazon GuardDuty**: 수집한 런타임 신호를 분석하고 위협 finding을 생성한다.
- **AWS Security Hub**: 보안 finding과 위협 분석을 중앙에서 확인하고 우선순위를 관리한다.

Runtime Monitoring은 예방 통제를 대체하지 않는다. 최소 권한 IAM, 이미지 취약점 검사, 네트워크 분리와 안전한 컨테이너 설정을 적용한 뒤에도 발생할 수 있는 비정상 행동을 탐지하는 계층이다.

## 서비스 및 아키텍처 관점

1. GuardDuty 보안 에이전트가 지원되는 EC2, EKS 또는 ECS on Fargate 워크로드의 런타임 활동을 수집한다.
2. GuardDuty가 운영체제, 네트워크와 파일 이벤트를 분석해 위협 finding을 생성한다.
3. Security Hub Threat Analytics 플랜이 finding과 관련 보안 신호를 중앙 보안 운영 흐름에 포함한다.
4. 보안 팀은 Security Hub에서 finding을 조사하고 필요한 경우 EventBridge와 자동 대응 흐름을 연결한다.
5. 사용량과 비용은 Security Hub의 단일 Runtime Monitoring 사용 유형으로 집계된다.

여러 계정 환경에서는 AWS Organizations의 위임 관리자 계정을 사용해 GuardDuty와 Security Hub를 중앙 관리하는 구성이 일반적이다. 워크로드 계정에서 생성된 finding을 보안 계정으로 집계하고, 심각도와 리소스 유형에 따라 조사 또는 자동 대응을 수행할 수 있다.

## 보안 및 운영 관점

- 조직의 모든 대상 계정과 리전에서 Security Hub 및 Runtime Monitoring 활성화 상태를 점검한다.
- 관리 계정 하나만 확인하지 말고 실제 워크로드가 있는 각 계정과 리전의 보호 범위를 검증한다.
- GuardDuty 보안 에이전트의 배포 상태와 권한을 모니터링해 탐지 공백이 생기지 않도록 한다.
- EKS와 ECS의 신규 클러스터 및 태스크가 자동으로 보호 범위에 들어오는지 온보딩 절차에 포함한다.
- finding의 심각도, 리소스, 계정과 리전을 기준으로 조사 우선순위 및 대응 책임자를 정한다.
- EventBridge 규칙, Lambda 또는 Systems Manager Automation으로 격리와 알림을 자동화할 때 오탐에 대한 승인 절차를 둔다.
- 런타임 탐지와 함께 Amazon Inspector 이미지 및 인스턴스 취약점 검사, ECR 이미지 스캔과 IAM 최소 권한을 적용한다.
- AWS Cost Explorer와 Security Hub 사용량 페이지에서 통합 이후 청구 변화를 확인한다.
- 기존 GuardDuty 청구가 사라졌다고 보호가 비활성화된 것으로 오해하지 말고 finding 및 에이전트 상태로 실제 탐지 범위를 확인한다.

## 비용 및 전환 시 주의점

- Security Hub가 활성화된 계정과 리전에서는 Runtime Monitoring 비용이 Security Hub 항목으로 청구된다.
- EC2, EKS와 ECS on Fargate가 개별 사용 유형이 아니라 하나의 사용 유형으로 측정된다.
- 기존 탐지 범위, finding 유형과 보안 에이전트는 변경되지 않는다.
- 별도의 재구성 작업은 필요하지 않다.
- Threat Analytics 플랜의 무료 평가판은 Security Hub Essentials 플랜의 무료 평가판과 별개다.
- 이번 변경으로 Runtime Monitoring의 새로운 무료 평가판이 추가되는 것은 아니다.

## 시험 관점 정리

- Amazon GuardDuty는 AWS 계정, 네트워크와 워크로드 신호를 분석해 위협을 탐지하는 관리형 서비스다.
- GuardDuty Runtime Monitoring은 EC2, EKS와 ECS on Fargate의 실행 중 활동을 관찰한다.
- AWS Security Hub는 여러 AWS 보안 서비스의 finding을 중앙에서 집계하고 보안 태세를 관리한다.
- Amazon Inspector는 주로 소프트웨어 취약점과 의도하지 않은 네트워크 노출을 평가하며 Runtime Monitoring과 역할이 다르다.
- Amazon EventBridge는 보안 finding을 이벤트로 받아 알림이나 자동 대응 워크플로를 시작할 수 있다.
- 다중 계정 보안 운영에서는 AWS Organizations와 위임 관리자를 사용해 중앙 관리를 구성할 수 있다.
- 예방, 탐지와 대응 통제를 함께 설계해야 defense in depth가 완성된다.

## 같이 보면 좋은 학습 포인트

- GuardDuty finding 유형과 심각도 분류
- EKS Runtime Monitoring과 보안 에이전트 배포 방식
- ECS on Fargate의 책임 공유 모델
- 컨테이너 탈출, 권한 상승과 cryptomining 공격 패턴
- Security Hub Essentials와 Threat Analytics 플랜의 역할
- AWS Organizations 기반 보안 계정 구조
- EventBridge 및 Systems Manager를 이용한 finding 자동 대응
- GuardDuty, Inspector, Macie와 Detective의 차이
- Cost Explorer를 이용한 보안 서비스 비용 분석

## 한 줄 정리

GuardDuty Runtime Monitoring이 Security Hub Threat Analytics에 포함되면서 EC2, EKS와 ECS on Fargate의 런타임 위협 탐지는 유지되고 사용량 및 비용 관리는 Security Hub로 통합된다.
