# Amazon Inspector, EC2 AMI 취약점 스캔 지원

## 공식 출처

- [EC2 Amazon Machine Images can now be scanned for vulnerabilities through Amazon Inspector](https://aws.amazon.com/about-aws/whats-new/2026/11/ec2-amis-scanned-through-inspector/)
- 발표일: 2026년 11월 2일

## 핵심 내용

Amazon Inspector가 Amazon EC2의 Amazon Machine Image(AMI)에 대한 취약점 스캔을 지원한다.

이제 AMI로 EC2 인스턴스를 실행하거나 별도의 소프트웨어를 설치하지 않고도 이미지에 포함된 소프트웨어 취약점을 확인할 수 있다. 보안 문제를 실행 중인 인스턴스에서 발견한 뒤 대응하는 대신, 여러 인스턴스의 원본이 되는 AMI 단계에서 먼저 찾아 수정할 수 있다.

AMI 스캔은 다음 두 가지 방식으로 사용할 수 있다.

- 지속 평가: AMI 카탈로그를 계속 모니터링해 새로 발견된 취약점을 확인
- 일회성 스캔: EC2 콘솔에서 특정 AMI를 대상으로 필요할 때 검사

계정 단위 또는 AWS Organizations 전체에서 AMI 스캔을 활성화할 수 있다. AWS Security Hub 사용자는 조직 전체에 AMI 스캔을 추가 기능으로 활성화하고 Inspector 결과를 Security Hub에서 받을 수 있다.

## 왜 중요한가

AMI는 운영체제, 라이브러리, 에이전트와 애플리케이션 구성 요소가 포함된 EC2 인스턴스의 시작점이다. 취약한 AMI가 배포되면 같은 문제를 가진 인스턴스가 여러 계정과 리전에 반복해서 생성될 수 있다.

기존에는 AMI를 실제 인스턴스로 실행한 뒤 스캔하거나 별도의 검사 환경과 에이전트를 관리해야 하는 경우가 있었다. Inspector의 AMI 스캔은 실행 전 단계에서 이미지 자체를 평가하므로 다음과 같은 효과가 있다.

- 취약점이 여러 인스턴스로 확산되기 전에 원본 이미지에서 발견
- 검사 전용 인스턴스와 추가 소프트웨어 관리 부담 감소
- 새 CVE가 공개될 때 기존 AMI 카탈로그를 지속적으로 재평가
- 조직 전체에 일관된 이미지 보안 기준 적용
- 배포 파이프라인에서 취약한 AMI의 배포와 공유를 조기에 차단

## 서비스 및 아키텍처 관점

AMI 스캔은 단순한 보고 기능을 넘어 이미지 생성, 승인, 배포를 잇는 공급망 보안 흐름에 포함할 수 있다.

일반적인 아키텍처 흐름은 다음과 같다.

1. EC2 Image Builder가 패치와 애플리케이션을 포함한 새 AMI를 생성한다.
2. Amazon Inspector가 AMI의 취약점을 스캔한다.
3. 보안 기준을 충족하지 못한 AMI는 배포 파이프라인에서 중단한다.
4. 기준을 통과한 AMI만 승인 계정과 리전에 배포한다.
5. Allowed AMIs 설정으로 계정에서 사용할 수 있는 이미지 기준을 적용한다.
6. AWS Organizations의 Declarative Policies로 기준을 조직 전체에 강제한다.
7. 이후 새 취약점이 공개되면 지속 평가를 통해 기존 AMI도 다시 확인한다.

EC2 Image Builder와 통합하면 이미지 생성 파이프라인에서 스캔을 수행하고 보안 기준에 맞지 않는 AMI의 배포를 제한할 수 있다. Allowed AMIs에는 Inspector 결과를 바탕으로 AMI가 충족해야 할 보안 기준을 지정할 수 있으며, Declarative Policies를 이용해 여러 계정에 일관되게 적용할 수 있다.

이 구조는 취약점 탐지와 실행 통제를 분리한다. Inspector가 위험을 찾아 결과를 제공하고, Image Builder 파이프라인과 Allowed AMIs가 해당 결과에 따라 이미지의 배포 및 사용 여부를 통제한다.

## 보안 관점

- 조직 전체에서 AMI 스캔을 활성화해 보안 사각지대를 줄인다.
- Inspector 결과의 심각도, 수정 가능 여부, 사용 범위를 기준으로 대응 우선순위를 정한다.
- Allowed AMIs를 사용해 기준 미달 이미지가 새 인스턴스의 기반으로 사용되지 않도록 제한한다.
- Image Builder 배포 전에 보안 게이트를 두어 취약한 AMI의 공유와 복제를 막는다.
- Security Hub로 결과를 중앙 수집해 다른 보안 결과와 함께 분석한다.
- AMI 생성, 공유, 스캔 설정 및 정책 변경 권한에는 최소 권한을 적용한다.
- CloudTrail을 사용해 Inspector 활성화와 이미지 정책 변경 이력을 감사한다.

취약점 스캔 결과가 없다는 사실이 이미지가 완전히 안전하다는 의미는 아니다. 구성 오류, 노출된 비밀정보, 애플리케이션 논리 취약점, 런타임 위협은 별도의 통제가 필요하다. AMI 스캔은 다층 방어 중 이미지 취약점 관리 계층으로 사용해야 한다.

## 운영 관점

지속 평가를 사용하면 AMI를 만들 당시에는 알려지지 않았던 취약점도 나중에 발견할 수 있다. 따라서 새 이미지의 최초 승인뿐 아니라 이미 승인된 AMI의 상태 변화에 대응하는 절차가 필요하다.

운영 시에는 다음 항목을 관리한다.

- 계정 및 조직별 AMI 스캔 활성화 상태
- AMI 소유자, 생성 파이프라인, 사용 계정과 현재 배포 범위
- 심각도별 Inspector 결과와 수정 기한
- 새 취약점 발견 시 AMI 승인 상태 재평가
- 패치된 새 AMI 생성과 이전 AMI 사용 중단 절차
- Allowed AMIs 보안 기준과 Declarative Policies 적용 범위
- Image Builder 파이프라인의 실패 및 배포 차단 기록
- Security Hub 중앙 결과와 담당 팀 알림
- Amazon Inspector AMI 스캔 요금

이미 실행 중인 인스턴스는 새 AMI를 만드는 것만으로 자동 교체되지 않는다. 패치된 AMI를 생성한 뒤 Auto Scaling 그룹, 시작 템플릿, 배포 파이프라인에서 새 버전을 사용하고 기존 인스턴스를 순차적으로 교체해야 한다.

## 시험 관점 정리

AWS 시험이나 아키텍처 문제에서는 다음 개념을 연결해서 기억할 수 있다.

- AMI는 EC2 인스턴스를 시작하기 위한 이미지 템플릿이다.
- Amazon Inspector는 취약점 관리 서비스이며 이제 AMI도 검사할 수 있다.
- AMI 스캔에는 지속 평가와 특정 이미지 대상 일회성 검사가 있다.
- 스캔을 위해 EC2 인스턴스를 실행하거나 추가 소프트웨어를 설치할 필요가 없다.
- AWS Security Hub는 여러 보안 서비스의 결과를 중앙에서 관리한다.
- EC2 Image Builder는 표준화된 이미지 생성과 배포 파이프라인을 제공한다.
- Allowed AMIs는 계정에서 사용할 수 있는 AMI에 보안 기준을 적용한다.
- AWS Organizations Declarative Policies는 조직 전체에 EC2 설정을 일관되게 강제하는 데 사용할 수 있다.
- 이미지 스캔과 실행 중 인스턴스 보호는 서로 보완하는 통제다.

## 같이 보면 좋은 학습 포인트

- Amazon Inspector의 EC2, ECR, Lambda 및 AMI 스캔 범위
- EC2 Image Builder 구성 요소와 이미지 파이프라인
- Allowed AMIs의 audit mode와 enabled mode
- AWS Organizations Declarative Policies
- AWS Security Hub 결과 집계와 자동 대응
- CVE, CVSS, 취약점 우선순위 결정 방법
- 시작 템플릿과 Auto Scaling 인스턴스 교체
- 골든 AMI 기반 운영 및 패치 전략

## 한 줄 정리

Amazon Inspector의 AMI 스캔으로 인스턴스를 실행하지 않고도 이미지 취약점을 지속적으로 확인하고, Image Builder와 Allowed AMIs를 통해 취약한 이미지의 배포를 사전에 차단할 수 있게 되었다.
