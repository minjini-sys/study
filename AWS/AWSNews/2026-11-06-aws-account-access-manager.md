# AWS Account Access Manager, 기존 IAM 역할과 중앙 집중식 인력 접근 결합

## 공식 출처

- [Introducing AWS account access manager: IAM role flexibility with centralized workforce access](https://aws.amazon.com/blogs/security/introducing-aws-account-access-manager-iam-role-flexibility-with-centralized-workforce-access/)
- [AWS Identity and Access Management streamlines assignment of IAM roles to workforce users with account access manager](https://aws.amazon.com/about-aws/whats-new/2026/08/aws-iam-aam/)
- 상세 가이드 게시일: 2026년 11월 5일
- 기능 출시일: 2026년 8월 10일

## 핵심 내용

AWS Identity and Access Management(IAM)의 Account Access Manager는 각 AWS 계정에 이미 존재하는 IAM 역할을 AWS IAM Identity Center의 인력 사용자와 그룹에 중앙에서 할당할 수 있게 한다.

기존 IAM 역할의 유연한 신뢰 정책, 태그, 경로와 권한 구성을 유지하면서도 IAM Identity Center를 단일 페더레이션 지점으로 사용할 수 있다. 여러 사용자가 하나의 IAM 역할을 공유하더라도 세션에 사용자 컨텍스트가 포함되어 실제 어떤 사용자가 어느 계정에서 어떤 역할을 사용했는지 추적할 수 있다.

Account Access Manager는 IAM Identity Center permission set을 대체해야 하는 기능이 아니다. 같은 사용자와 그룹을 이용해 두 방식을 함께 사용할 수 있다.

- Permission set: 여러 계정에 표준화된 권한을 중앙 프로비저닝할 때 적합
- Account Access Manager: 계정별로 다르게 구성된 기존 IAM 역할을 그대로 중앙 할당할 때 적합

## 왜 중요한가

멀티 계정 환경의 인력 접근 관리에는 일반적으로 두 가지 접근이 있었다.

- 각 계정에서 IAM 역할과 외부 자격 증명 공급자 페더레이션을 따로 관리
- IAM Identity Center permission set으로 중앙에서 표준 권한을 프로비저닝

첫 번째 방식은 IAM 역할을 세밀하게 구성할 수 있지만 계정마다 페더레이션과 할당을 관리해야 한다. 두 번째 방식은 중앙 관리가 쉽지만, 계정마다 역할 구성이 다르거나 기존 역할을 그대로 유지해야 하는 환경에서는 표준화가 어려울 수 있다.

Account Access Manager는 기존 IAM 역할을 IAM Identity Center 사용자와 그룹에 직접 연결해 다음 효과를 제공한다.

- 계정별 기존 IAM 역할을 재설계하지 않고 중앙 할당
- 조직 전체의 단일 인력 로그인 지점
- 역할 세션에 실제 사용자 컨텍스트 유지
- IAM 역할의 ABAC, 태그, 경로, 신뢰 정책 활용
- 웹 콘솔과 `aws login`을 통한 임시 자격 증명 사용
- API와 인프라 코드를 이용한 반복 가능한 접근 할당

## 서비스 및 아키텍처 관점

기본적인 접근 흐름은 다음과 같다.

1. 조직 인스턴스 형태로 IAM Identity Center를 활성화한다.
2. 외부 IdP의 사용자와 그룹을 SCIM으로 동기화하거나 Identity Center 디렉터리에서 관리한다.
3. 각 AWS 계정에 업무에 필요한 IAM 역할을 준비한다.
4. 역할의 신뢰 정책이 해당 Account Access Manager 애플리케이션을 신뢰하도록 구성한다.
5. 사용자 또는 그룹, AWS 계정 ID, IAM 역할 ARN을 결합한 entitlement를 생성한다.
6. 사용자가 Account Access Portal 또는 IAM Identity Center Access Portal에서 계정과 역할을 선택한다.
7. CLI 사용자는 `aws login`으로 브라우저 인증 후 임시 자격 증명을 받는다.
8. 역할 세션의 사용자 컨텍스트를 이용해 실제 사용자의 활동을 감사한다.

Account Access Manager는 AWS Management Console, AWS CLI의 `account-access` API namespace, AWS SDK, AWS CloudFormation, Terraform과 Pulumi 등을 통해 관리할 수 있다.

역할 할당은 다음 세 요소로 구성된다.

- IAM Identity Center 사용자 또는 그룹 ID
- 대상 AWS 계정 ID
- 대상 계정에 존재하는 IAM 역할 ARN

조직의 관리 계정에서 기능을 활성화하고 IAM Identity Center 인스턴스의 기본 리전과 맞춰야 한다. 일상적인 할당 관리는 위임 관리와 자동화를 이용해 관리 계정의 직접 사용을 줄이는 것이 좋다.

## 보안 관점

- 역할 신뢰 정책은 Account Access Manager 서비스와 자신의 애플리케이션 ARN으로 범위를 제한해 confused deputy 위험을 줄인다.
- 역할은 개별 사용자보다 그룹에 할당해 입사, 이동, 퇴사에 따른 권한 변경을 단순화한다.
- 기능 관리 권한과 entitlement 관리 권한을 분리하고 최소 권한을 적용한다.
- ABAC가 필요할 때만 신뢰 정책에 `sts:TagSession`을 허용하고 허용할 태그 키와 값의 출처를 통제한다.
- IAM 역할의 최대 세션 기간을 업무 위험에 맞게 설정한다. Account Access Manager 세션은 기반 역할의 최대 세션 기간을 사용하며 최대 12시간까지 가능하다.
- CloudTrail에서 별도의 `account-access` API 이벤트와 역할 사용 기록을 감사한다.
- 방화벽 또는 Secure Web Gateway를 사용하는 경우 Account Access Portal과 control plane, runtime endpoint를 사전에 허용한다.
- IAM Identity Center가 고객 관리형 KMS 키로 암호화된 경우 Account Access Manager가 필요한 데이터를 읽을 수 있도록 키 정책을 검토한다.

Account Access Manager가 사용자 컨텍스트를 제공하더라도 역할 자체의 권한이 과도하면 위험은 그대로 남는다. 역할 권한 정책, 신뢰 정책, Organizations SCP, permissions boundary와 세션 정책을 함께 검토해야 한다.

## 운영 관점

기존 역할을 그대로 사용할 수 있다는 장점은 역할 구성이 계정마다 달라질 수 있다는 의미이기도 하다. 역할 이름, 경로, 태그, 신뢰 정책과 권한 정책의 표준을 정하고 인프라 코드로 관리하는 것이 중요하다.

운영 시에는 다음 항목을 관리한다.

- IAM Identity Center 조직 인스턴스와 기본 리전
- Account Access Manager 애플리케이션 ARN
- 할당 가능한 IAM 역할과 신뢰 정책 상태
- 사용자·그룹·계정·역할로 구성된 entitlement 목록
- 그룹 멤버십과 퇴사자 접근 회수
- 역할 최대 세션 기간과 임시 자격 증명 갱신
- `aws login`에 필요한 AWS CLI 버전과 개발자 안내
- CloudTrail 기반 사용자 및 역할 활동 감사
- IaC를 통한 entitlement 변경 검토와 배포

사용자 세션은 접근 할당을 삭제한 직후 즉시 종료되지 않고 기존 세션 만료 시점까지 유지될 수 있다. 긴급 권한 회수 절차에서는 entitlement 삭제뿐 아니라 활성 세션과 기반 IAM 역할의 대응도 함께 고려해야 한다.

Account Access Manager는 추가 비용 없이 제공되며 기본 활성화 상용 리전에서 사용할 수 있다. 옵트인 상용 리전, AWS GovCloud(US), 중국 및 비상용 리전은 공식 가용성 범위를 별도로 확인해야 한다.

## 시험 관점 정리

AWS 시험이나 아키텍처 문제에서는 다음 개념을 연결해서 기억할 수 있다.

- IAM 역할은 계정 안에서 권한 정책과 신뢰 정책을 분리해 관리한다.
- IAM Identity Center는 인력 사용자의 AWS 계정과 애플리케이션 접근을 중앙 관리한다.
- Account Access Manager는 기존 IAM 역할을 Identity Center 사용자와 그룹에 할당한다.
- Permission set과 Account Access Manager는 함께 사용할 수 있다.
- Trusted identity propagation은 공유 역할 세션에서도 실제 사용자 컨텍스트를 전달한다.
- 장기 액세스 키 대신 페더레이션과 임시 자격 증명을 사용한다.
- ABAC는 principal tag와 리소스 태그 같은 속성에 따라 접근을 제어한다.
- CloudTrail은 역할 수임과 API 활동을 감사하는 데 사용한다.

## 같이 보면 좋은 학습 포인트

- IAM 역할의 권한 정책과 신뢰 정책
- IAM Identity Center permission set
- SAML, SCIM과 기업 IdP 연동
- Trusted identity propagation
- ABAC와 `aws:PrincipalTag`
- AWS Organizations SCP와 위임 관리자
- AWS CLI `aws login`과 임시 자격 증명
- CloudTrail의 사용자 신원 및 역할 세션 분석

## 한 줄 정리

AWS Account Access Manager는 계정별 기존 IAM 역할의 유연성을 유지하면서 IAM Identity Center를 통해 사용자와 그룹에 중앙 할당하고 실제 사용자 단위의 접근 추적을 가능하게 한다.
