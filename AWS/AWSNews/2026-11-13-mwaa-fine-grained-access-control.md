# Amazon MWAA, 세분화 접근 제어와 다중 팀 환경 지원

## 공식 출처

- [Amazon MWAA now supports fine-grained access control with IAM Identity Center and multi-team environments](https://aws.amazon.com/about-aws/whats-new/2026/11/mwaa-fine-grained-access-control/)
- 발표일: 2026년 11월 12일

## 핵심 내용

Amazon Managed Workflows for Apache Airflow(Amazon MWAA)가 AWS IAM Identity Center 연동과 다중 팀 환경을 통해 세분화된 접근 제어를 지원한다. 관리자는 하나의 Airflow 환경에서 사용자와 팀이 접근할 수 있는 워크플로 및 리소스를 더 정확하게 제한할 수 있다.

새 기능은 크게 두 부분으로 나뉜다.

- IAM Identity Center 연동: 사용자가 기업의 자격 증명 공급자(IdP)로 Airflow UI에 로그인한다.
- 요청별 권한 평가: IAM Identity Center Permission Set에 연결된 IAM 정책을 사용해 요청마다 권한을 판단한다.
- 조건 키 기반 범위 제한: DAG, 연결, 변수, 풀 등 리소스 유형과 작업, 특정 리소스 ID를 기준으로 접근을 제한한다.
- 다중 팀 환경: 여러 팀이 하나의 MWAA 환경을 공유할 수 있다.
- 팀별 DAG 분리: 각 팀은 Amazon S3 안에 전용 DAG 폴더를 가진다.
- 그룹 기반 표시: 사용자는 IAM Identity Center 그룹 멤버십에 따라 공용 DAG와 자신의 팀 DAG만 볼 수 있다.
- 모든 Amazon MWAA 지원 리전에서 사용할 수 있다.

다중 팀 환경은 현재 미리 보기 상태인 Apache Airflow의 multi-team 기능을 기반으로 한다. 따라서 실제 운영에 적용하기 전 지원 범위와 변경 가능성을 확인해야 한다.

## 왜 중요한가

여러 데이터 팀이 Airflow를 사용할 때 팀마다 별도 MWAA 환경을 만들면 격리는 명확하지만 환경 비용, 패치, 네트워크와 공통 설정 관리가 반복된다. 반대로 하나의 환경을 단순 공유하면 다른 팀의 DAG, 연결 정보나 운영 리소스에 불필요하게 접근할 위험이 있다.

이번 기능은 공유 인프라의 비용 효율성과 팀별 접근 격리를 함께 제공한다. 기업 IdP와 IAM Identity Center 그룹을 기준으로 사용자를 관리하고, 요청 단위 IAM 정책으로 리소스 접근 범위를 제한할 수 있다. 플랫폼 팀은 환경 설정을 한 번 관리하면서 각 팀에는 필요한 워크플로만 보이도록 구성할 수 있다.

## 서비스 및 아키텍처 관점

일반적인 구성에서는 기업 IdP의 사용자와 그룹을 IAM Identity Center에 연결하고, Permission Set의 IAM 정책에 Airflow 리소스와 작업 범위를 정의한다. MWAA는 사용자의 각 요청을 이 정책으로 평가한다.

다중 팀 환경의 접근 흐름은 다음과 같이 이해할 수 있다.

1. 사용자가 기업 IdP를 통해 IAM Identity Center로 인증한다.
2. IAM Identity Center 그룹과 Permission Set이 사용자의 권한 범위를 결정한다.
3. 사용자가 MWAA의 Airflow UI에 접속한다.
4. 각 요청은 리소스 유형, 작업 및 리소스 ID 조건을 포함한 IAM 정책으로 평가된다.
5. 사용자는 공용 DAG와 자신이 속한 팀의 S3 DAG 폴더에 있는 DAG만 확인한다.
6. 여러 팀은 같은 MWAA 실행 환경과 공통 설정을 공유한다.

공유 환경은 비용과 관리 부담을 낮추지만 컴퓨팅 자원, 스케줄러와 장애 영향 범위도 공유한다. 접근 제어와 실행 자원 격리는 서로 다른 문제이므로 팀별 부하, 중요도와 장애 격리 요구사항에 따라 별도 환경이 더 적합할 수 있다.

## 보안 관점

- 기업 IdP와 IAM Identity Center를 사용해 사용자 생명 주기와 로그인을 중앙 관리한다.
- Permission Set 정책에는 필요한 Airflow 작업과 리소스만 허용하는 최소 권한을 적용한다.
- 리소스 유형뿐 아니라 특정 DAG, 연결, 변수, 풀의 ID까지 조건 키로 제한한다.
- IAM Identity Center 그룹과 MWAA 팀의 매핑을 정기적으로 검토한다.
- 팀별 S3 DAG 경로에 대한 쓰기 권한을 분리해 다른 팀의 워크플로 변경을 방지한다.
- Airflow 연결과 변수에 비밀 정보를 직접 저장할 때의 노출 위험을 검토하고 AWS Secrets Manager 같은 비밀 관리 서비스를 고려한다.
- CloudTrail, MWAA 로그와 IdP 감사 로그를 연결해 로그인과 권한 변경을 추적한다.
- 공용 DAG는 여러 팀에 영향을 주므로 별도의 코드 검토와 배포 통제를 적용한다.

UI에 DAG가 보이지 않는 것만으로 완전한 보안 경계가 만들어지는 것은 아니다. S3, IAM 역할, 네트워크, 비밀 정보와 Airflow 실행 권한을 함께 검토해야 한다.

## 운영 관점

운영 시에는 다음 항목을 관리한다.

- IdP 그룹, IAM Identity Center 그룹과 MWAA 팀의 매핑
- Permission Set 정책과 조건 키의 정확성
- 팀별 및 공용 S3 DAG 폴더 구조와 배포 파이프라인
- 권한 거부 오류와 예상하지 못한 접근 허용
- 팀별 DAG 실행량, 큐 지연과 작업 실패율
- 공유 환경의 스케줄러 및 워커 용량
- 한 팀의 과도한 작업이 다른 팀에 미치는 영향
- 공통 Airflow 설정과 플러그인 변경 절차
- Apache Airflow multi-team 미리 보기 기능의 지원 상태와 변경 사항

도입 전에는 대표 사용자 계정으로 팀별 허용·거부 테스트를 수행해야 한다. 운영 환경에서는 정기적인 접근 권한 검토와 함께 팀별 비용 및 자원 사용량을 관찰해 공유 환경 유지 여부를 판단하는 것이 좋다.

## 시험 관점 정리

AWS 시험이나 아키텍처 문제에서는 다음 개념을 연결해서 기억할 수 있다.

- Amazon MWAA는 Apache Airflow를 위한 관리형 워크플로 오케스트레이션 서비스다.
- IAM Identity Center 연동을 통해 기업 IdP 사용자로 Airflow UI에 로그인할 수 있다.
- 권한은 Permission Set에 연결된 IAM 정책을 사용해 요청마다 평가된다.
- 조건 키로 DAG, 연결, 변수, 풀 등의 리소스 유형, 작업과 리소스 ID를 제한할 수 있다.
- 다중 팀 환경에서는 여러 팀이 하나의 MWAA 환경을 공유한다.
- 각 팀은 S3의 전용 DAG 폴더를 사용하고 그룹 멤버십에 따라 DAG 가시성이 결정된다.
- 공유 환경은 비용과 관리 부담을 줄이지만 성능 자원과 장애 영향 범위를 공유한다.
- Apache Airflow multi-team 기능은 현재 미리 보기이므로 운영 적용 전 제한 사항을 확인해야 한다.

## 같이 보면 좋은 학습 포인트

- Amazon MWAA와 직접 운영하는 Apache Airflow 비교
- IAM Identity Center Permission Set
- IAM 정책의 조건 키와 리소스 수준 권한
- 기업 IdP와 SAML 또는 SCIM 연동
- Airflow DAG, Connection, Variable과 Pool
- S3 접두사 기반 팀별 DAG 배포
- Airflow 비밀 백엔드와 AWS Secrets Manager
- 멀티 테넌트 환경의 격리 및 noisy neighbor 문제

## 한 줄 정리

Amazon MWAA의 IAM Identity Center 연동과 다중 팀 환경은 기업 사용자에게 요청 단위 최소 권한을 적용하면서 여러 팀이 하나의 Airflow 환경을 안전하고 효율적으로 공유하도록 돕는다.
