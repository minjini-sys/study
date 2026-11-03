# Amazon EKS Capabilities, Argo CD Notifications 지원

## 공식 출처

- [Amazon EKS Capabilities now supports Argo CD Notifications](https://aws.amazon.com/about-aws/whats-new/2026/11/amazon-eks-argo-cd-notifications/)
- 발표일: 2026년 11월 3일

## 핵심 내용

Amazon EKS Capability for Argo CD가 Argo CD Notifications를 지원한다.

이제 Argo CD가 관리하는 애플리케이션에서 동기화가 발생하거나, 배포가 실패하거나, 상태가 변경될 때 알림을 보낼 수 있다. 트리거와 메시지 템플릿은 클러스터에 정의하고, 각 Argo CD Application에 annotation을 추가해 필요한 알림을 구독한다. 이는 업스트림 Argo CD Notifications와 같은 구성 방식이다.

알림은 Slack, PagerDuty, Opsgenie, Amazon SQS, Amazon EventBridge를 포함한 16개 서비스로 전달할 수 있다.

AWS 서비스와 외부 서비스의 인증 방식은 다음처럼 구분된다.

- Amazon SQS와 Amazon EventBridge: IAM Capability Role로 인증
- Slack 토큰처럼 외부 서비스 자격 증명이 필요한 경우: AWS Secrets Manager ARN을 지정하고 EKS가 대신 사용

## 왜 중요한가

GitOps에서는 Git 저장소의 선언 상태와 클러스터의 실제 상태를 Argo CD가 지속적으로 비교하고 동기화한다. 그러나 동기화 실패나 애플리케이션 상태 악화를 담당자가 빠르게 알지 못하면 배포 오류가 오래 지속될 수 있다.

Argo CD Notifications 지원으로 다음과 같은 운영 개선이 가능하다.

- 배포 성공과 실패를 개발 및 운영 채널에 즉시 전달
- 애플리케이션 상태 변화에 따른 온콜 호출
- SQS 또는 EventBridge를 통한 후속 자동화 실행
- 여러 팀이 각 Application 단위로 필요한 알림만 구독
- 클러스터 내부에 AWS 자격 증명을 직접 저장하지 않는 서비스 연동
- 외부 서비스 토큰을 Secrets Manager에서 중앙 관리

## 서비스 및 아키텍처 관점

일반적인 알림 흐름은 다음과 같다.

1. 개발자가 Git 저장소의 Kubernetes 매니페스트를 변경한다.
2. Argo CD가 변경을 감지하고 EKS 클러스터와 동기화한다.
3. 애플리케이션의 동기화, 실패 또는 상태 변경 이벤트가 발생한다.
4. Argo CD Notifications가 정의된 트리거 조건을 평가한다.
5. 메시지 템플릿을 이용해 알림 내용을 생성한다.
6. Application annotation에 지정된 구독 대상에 알림을 전달한다.
7. 사람 또는 이벤트 기반 워크플로가 후속 조치를 수행한다.

알림 대상을 목적에 따라 나눌 수 있다.

- Slack: 팀이 배포 상태를 빠르게 공유
- PagerDuty 또는 Opsgenie: 심각한 실패를 온콜 담당자에게 전달
- Amazon SQS: 이벤트를 버퍼링하고 소비자가 안정적으로 후속 처리
- Amazon EventBridge: 규칙을 이용해 Lambda, Step Functions 등 여러 대상으로 라우팅

SQS와 EventBridge는 IAM Capability Role을 사용하므로 장기 AWS 액세스 키를 Kubernetes Secret에 넣을 필요가 없다. 이를 통해 Argo CD 배포 이벤트를 AWS의 이벤트 기반 아키텍처와 자연스럽게 연결할 수 있다.

## 보안 관점

- IAM Capability Role에는 필요한 SQS 큐 또는 EventBridge 대상에 대한 최소 권한만 부여한다.
- 리소스 ARN과 조건을 사용해 다른 큐나 이벤트 버스로의 임의 전송을 제한한다.
- Slack 같은 외부 서비스의 토큰은 평문 Kubernetes Secret 대신 Secrets Manager ARN으로 참조한다.
- Secrets Manager 비밀 값을 읽을 수 있는 역할과 리소스 정책을 최소 범위로 제한한다.
- 알림 메시지에 환경 변수, Secret, 고객 데이터 등 민감한 정보가 포함되지 않도록 템플릿을 검토한다.
- CloudTrail을 이용해 IAM 역할과 Secrets Manager 접근 이력을 감사한다.
- Application annotation 변경 권한을 제한해 승인되지 않은 외부 대상으로 알림이 전송되지 않도록 한다.

알림 기능 자체가 배포 권한을 부여하는 것은 아니지만, 메시지에 내부 시스템 정보가 포함되거나 이벤트가 자동 조치로 이어질 수 있다. 따라서 알림 대상과 후속 자동화의 권한 경계를 함께 설계해야 한다.

## 운영 관점

모든 이벤트를 모든 채널로 보내면 알림 피로가 생길 수 있다. 성공 알림은 팀 채널, 반복 실패나 상태 악화는 온콜 서비스, 자동 처리가 필요한 이벤트는 EventBridge처럼 목적별로 분리하는 것이 좋다.

운영 시에는 다음 항목을 관리한다.

- 트리거 조건과 메시지 템플릿의 버전 관리
- Application annotation 기반 구독 상태
- 환경과 심각도에 따른 알림 대상 분리
- SQS의 적체, 재처리 및 Dead-Letter Queue 구성
- EventBridge 규칙의 일치 여부와 대상 전달 실패
- 외부 서비스 토큰의 만료와 교체
- 중복 이벤트에 대한 후속 처리의 멱등성
- 알림 전달 실패에 대한 별도 모니터링
- 사용 중인 알림 서비스의 표준 요금

Argo CD Notifications 기능에는 추가 요금이 없지만 SQS, EventBridge 등 연결된 서비스에는 표준 요금이 적용된다. 알림량과 보존, 후속 Lambda 또는 Step Functions 실행 비용까지 함께 확인해야 한다.

## 시험 관점 정리

AWS 시험이나 아키텍처 문제에서는 다음 개념을 연결해서 기억할 수 있다.

- GitOps는 Git의 선언 상태를 기준으로 애플리케이션 배포 상태를 관리한다.
- Argo CD는 Kubernetes 애플리케이션의 지속적 배포와 상태 동기화를 담당한다.
- 트리거는 언제 알림을 보낼지, 템플릿은 어떤 내용을 보낼지 정의한다.
- Application annotation을 사용해 개별 애플리케이션을 알림에 구독시킬 수 있다.
- SQS는 이벤트 버퍼링과 비동기 소비에 적합하다.
- EventBridge는 이벤트 필터링과 여러 AWS 서비스로의 라우팅에 적합하다.
- IAM 역할은 장기 액세스 키보다 안전한 AWS 서비스 인증 방법이다.
- 외부 서비스 자격 증명은 Secrets Manager에서 중앙 관리할 수 있다.

## 같이 보면 좋은 학습 포인트

- GitOps와 Argo CD의 동기화 모델
- Argo CD Application과 ApplicationSet
- Argo CD Notifications의 trigger, template, subscription
- Amazon EKS Capabilities와 IAM Capability Role
- Amazon EventBridge 규칙 및 이벤트 패턴
- Amazon SQS와 Dead-Letter Queue
- AWS Secrets Manager의 비밀 교체와 리소스 정책
- 배포 실패 자동 롤백 및 ChatOps 패턴

## 한 줄 정리

EKS Capability for Argo CD의 Notifications 지원으로 GitOps 배포 상태를 16개 서비스에 전달하고, IAM 역할과 Secrets Manager를 이용해 자격 증명을 클러스터에 직접 저장하지 않는 알림 자동화를 구성할 수 있게 되었다.
