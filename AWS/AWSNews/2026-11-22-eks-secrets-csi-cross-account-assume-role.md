# EKS Secrets Store CSI Driver, 교차 계정 AssumeRole 지원

- 정리 날짜: 2026-11-22
- 공식 발표일: 2026-11-20
- 공식 발표: [AWS Secrets Store CSI Driver Provider now supports AssumeRole for cross-account secret access](https://aws.amazon.com/about-aws/whats-new/2026/11/secrets-store-csi-driver-assumerole/)
- 공식 문서: [Assume an IAM role to retrieve secrets with the ASCP](https://docs.aws.amazon.com/secretsmanager/latest/userguide/integrating_ascp_assume_role.html)

## 핵심 내용

AWS Secrets Store CSI Driver Provider가 `AssumeRole` 기반 역할 체이닝을 지원한다. 이제 Amazon EKS 파드는 시크릿이 저장된 다른 AWS 계정의 IAM 역할을 수임해 AWS Secrets Manager 시크릿과 AWS Systems Manager Parameter Store 파라미터를 가져올 수 있다.

사용자는 Kubernetes `SecretProviderClass`의 `assumeRoleArn`에 대상 계정의 IAM 역할 ARN을 지정한다. AWS Secrets and Configuration Provider(ASCP)는 파드가 IRSA 또는 EKS Pod Identity로 받은 기본 역할 자격 증명을 사용해 AWS STS `AssumeRole`을 호출하고, 대상 역할의 임시 자격 증명으로 시크릿을 읽어 파드 볼륨에 마운트한다. 대상 역할의 자격 증명은 ASCP만 사용하며 애플리케이션 컨테이너의 기본 자격 증명은 바뀌지 않는다.

이 기능은 초기 마운트, 시크릿 순환 폴링 및 장애 조치 리전 구성에서 IRSA와 EKS Pod Identity를 모두 지원한다. Amazon EKS와 AWS Secrets Store CSI Driver Provider가 제공되는 모든 AWS 리전에서 사용할 수 있다.

## 왜 중요한가

기존의 직접적인 교차 계정 접근 방식은 각 Secrets Manager 시크릿에 리소스 정책을 추가하고, 교차 계정에서 사용할 수 없는 AWS 관리형 키 `aws/secretsmanager` 대신 고객 관리형 AWS KMS 키와 키 정책을 구성해야 했다. 시크릿이 많고 계정이 수백 개라면 시크릿별 정책 관리가 복잡해지고 누락 가능성도 커진다.

새 방식은 시크릿 소유 계정에 대상 역할 하나를 만들고 신뢰 정책과 권한 정책을 중앙 관리할 수 있다. 파드는 그 역할을 수임한 뒤 소유 계정 내부의 권한으로 시크릿을 읽으므로 시크릿마다 교차 계정 리소스 정책을 추가할 필요가 없고, `aws/secretsmanager`로 암호화된 시크릿도 사용할 수 있다.

또한 역할 세션 이름에 Kubernetes 네임스페이스와 서비스 계정 정보가 반영되어 대상 계정의 CloudTrail에서 어떤 워크로드가 시크릿을 읽었는지 추적하기 쉬워진다.

## 서비스·아키텍처 관점

접근 흐름은 다음과 같이 이해할 수 있다.

1. EKS 파드가 IRSA 또는 EKS Pod Identity로 기본 역할 자격 증명을 받는다.
2. ASCP가 `SecretProviderClass`의 `assumeRoleArn`을 확인한다.
3. 기본 역할로 대상 계정 역할에 `sts:AssumeRole`을 호출한다.
4. ASCP가 대상 역할의 임시 자격 증명으로 Secrets Manager 또는 Parameter Store를 조회한다.
5. 조회한 값을 CSI 볼륨의 파일로 파드에 마운트한다.

멀티 계정 환경에서는 애플리케이션 계정에 EKS 클러스터를 두고 보안 또는 공유 서비스 계정에 시크릿을 중앙 저장하는 구조에 활용할 수 있다. 같은 Kubernetes 서비스 계정을 쓰는 여러 `SecretProviderClass`에도 서로 다른 대상 역할을 지정할 수 있어 워크로드별 권한 경계를 나눌 수 있다.

ASCP는 마운트와 순환 폴링 때마다 `AssumeRole`을 호출한다. 따라서 대규모 파드 확장이나 짧은 순환 간격을 사용할 때는 클러스터 계정의 AWS STS 요청 할당량과 마운트 지연을 고려해야 한다. ASCP 기반 CSI 마운트는 Amazon EKS의 EC2 노드 그룹에서 사용하며 EKS Fargate 노드 그룹은 지원하지 않는다.

## 보안 관점

권한은 기본 역할, 대상 역할, 대상 역할의 신뢰 정책에 나누어 최소 권한으로 구성해야 한다.

- 기본 역할: 지정한 대상 역할에 대한 `sts:AssumeRole`
- EKS Pod Identity 기본 역할: 세션 태그를 사용할 경우 `sts:TagSession`도 필요
- 대상 역할: 필요한 시크릿에 대한 `secretsmanager:GetSecretValue`, `secretsmanager:DescribeSecret`
- Parameter Store 사용 시: 필요한 파라미터에 대한 `ssm:GetParameters`
- 고객 관리형 KMS 키 사용 시: 해당 키에 대한 `kms:Decrypt`
- 대상 역할 신뢰 정책: 정확한 기본 역할만 수임하도록 허용

EKS Pod Identity의 세션 태그를 사용하면 대상 역할의 신뢰 정책 조건에서 클러스터 ARN, Kubernetes 네임스페이스, 서비스 계정을 제한할 수 있다. 이 조건을 활용하면 같은 기본 역할을 공유하더라도 의도한 워크로드만 대상 역할을 수임하도록 방어 계층을 추가할 수 있다.

ASCP는 외부 ID를 보내지 않으므로 신뢰 정책의 주체와 조건을 엄격하게 지정해야 한다. CloudTrail의 `AssumeRole` 및 시크릿 조회 이벤트를 중앙 수집하고 비정상적인 역할 수임이나 조회 패턴에 경보를 설정하는 것도 중요하다.

## 운영 관점

- `SecretProviderClass`에 올바른 `assumeRoleArn`과 리전을 명시한다.
- 기본 역할의 권한 정책과 대상 역할의 신뢰 정책을 함께 검토한다.
- Pod Identity에서 세션 태그를 사용한다면 양쪽 정책에 `sts:TagSession`이 있는지 확인한다.
- 마운트 실패 시 `kubectl describe pod`의 `FailedMount` 이벤트에서 STS 오류와 두 역할 ARN을 확인한다.
- 대상 계정의 CloudTrail에서 `namespace@serviceAccountName` 형식의 역할 세션 이름으로 호출 워크로드를 추적한다.
- 자동 순환을 사용할 때 애플리케이션이 변경된 마운트 파일을 다시 읽는지 검증한다. Kubernetes Secret과 환경 변수로 동기화했다면 파드 재시작이 필요할 수 있다.
- 다수의 파드가 동시에 시작하거나 순환 폴링하는 환경에서는 STS 호출량과 API 제한을 모니터링한다.
- `failoverRegion`을 사용한다면 보조 리전의 역할, 시크릿, KMS 권한도 함께 시험한다.

## 시험 관점 정리

AWS 자격시험에서는 다음 연결 관계를 기억하면 좋다.

- **IRSA**: Kubernetes 서비스 계정을 IAM 역할과 연결하는 OIDC 기반 방식
- **EKS Pod Identity**: EKS가 관리하는 파드 단위 IAM 자격 증명 연결 방식
- **AWS STS AssumeRole**: 다른 역할의 임시 보안 자격 증명을 발급받는 기능
- **Secrets Store CSI Driver**: 외부 시크릿을 Kubernetes 파드의 볼륨 파일로 마운트
- **ASCP**: Secrets Manager와 Parameter Store의 값을 CSI Driver에 제공하는 AWS 공급자
- **교차 계정 최소 권한**: 기본 역할에는 역할 수임 권한을, 대상 역할에는 시크릿 조회 권한을 분리

시크릿별 리소스 정책과 고객 관리형 KMS 키 없이 여러 EKS 워크로드가 중앙 시크릿 계정에 접근해야 하는 문제라면, `assumeRoleArn`을 사용하는 대상 역할 방식이 적합한 선택지가 될 수 있다.

## 같이 보면 좋은 학습 포인트

1. IRSA와 EKS Pod Identity의 인증 흐름 및 차이
2. IAM 역할의 권한 정책과 신뢰 정책의 역할 차이
3. STS 역할 체이닝, 세션 태그 및 CloudTrail 추적 방식
4. Secrets Manager 리소스 정책과 KMS 키 정책을 이용한 기존 교차 계정 접근
5. CSI 볼륨 마운트와 Kubernetes Secret 동기화의 보안·운영 차이
6. Secrets Manager 자동 순환과 CSI Driver rotation reconciler

## 한 줄 정리

AWS Secrets Store CSI Driver Provider의 AssumeRole 지원은 EKS 파드가 대상 계정의 중앙 IAM 역할을 통해 교차 계정 시크릿을 읽게 해, 시크릿별 리소스 정책을 줄이고 최소 권한과 감사 추적을 단순화한다.
