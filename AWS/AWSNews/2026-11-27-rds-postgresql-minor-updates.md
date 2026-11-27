# Amazon RDS for PostgreSQL, 새 마이너 버전 5종 지원

- 정리일 2026-11-27
- 발표일 2026-11-26
- [AWS What's New 공식 발표](https://aws.amazon.com/about-aws/whats-new/2026/11/amazon-rds-postgresql-18-7-17-12-16-16-15-20-14-25/)

## 핵심 내용

Amazon RDS for PostgreSQL에서 18.7, 17.12, 16.16, 15.20, 14.25를 지원한다. AWS는 이전 버전의 보안 취약점과 버그를 해결하기 위해 현재 메이저 버전에 해당하는 최신 마이너 버전으로 업데이트할 것을 권고했다.

## 왜 중요한가

PostgreSQL 16을 사용하는 팀은 16.16을 검토하면서 메이저 버전 전환에 필요한 애플리케이션 마이그레이션과 별개로 보안 패치 일정을 잡을 수 있다. 여러 계정에 DB가 있다면 개발·검증 환경에서 애플리케이션의 연결과 주요 쿼리를 확인한 뒤 운영 환경으로 순차 적용할 수 있다. [AWS Organizations 업그레이드 롤아웃 정책](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_upgrade_rollout.html)

## 서비스·아키텍처 관점

정기 유지 관리 창을 활용할 수 있는 DB라면 자동 마이너 업그레이드를 검토한다. 중단 시간을 줄여야 하는 워크로드는 지원되는 구성에서 RDS Blue/Green Deployments로 새 버전을 미리 검증한 뒤 서비스 전환을 계획할 수 있다. RDS for PostgreSQL은 마이너 업그레이드용 Blue/Green 구성에서 물리 복제를 사용한다. [RDS PostgreSQL Blue/Green 복제 방식](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/blue-green-deployments-replication-type.html)

## 보안 관점

현재 엔진 버전을 인벤토리로 확인하고 보안 수정이 필요한 DB의 패치 일정을 잡는다. 엔진 업그레이드와 확장 모듈 업그레이드는 별도 작업이므로 설치된 확장의 버전과 호환성도 점검한다. [RDS for PostgreSQL 자동 마이너 업그레이드](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_UpgradeDBInstance.PostgreSQL.Minor.html)

## 운영 관점

- 대상 리전과 현재 엔진 버전을 기준으로 지원되는 업그레이드 경로를 확인한다. 자동 적용은 RDS가 해당 리전과 메이저 버전에 지정한 마이너 버전을 대상으로 한다. [자동 업그레이드 대상 확인](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_UpgradeDBInstance.PostgreSQL.Minor.html)
- 자동 업그레이드가 활성화된 여러 계정을 운영한다면 AWS Organizations 롤아웃 정책으로 검증 환경과 운영 환경의 적용 시점을 단계화한다. [업그레이드 롤아웃 정책](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_upgrade_rollout.html)
- 변경 전에 복구 지점과 애플리케이션 재연결 절차를 확인하고, 변경 뒤에는 연결 오류율과 주요 쿼리 지연을 비교한다.

## 시험 관점

유지 관리 창은 자동 마이너 업그레이드의 적용 시점을 제어하고, AWS Organizations 정책은 여러 계정의 적용 순서를 조율한다. Blue/Green Deployments는 별도 환경에서 변경을 검증하고 전환 시점의 중단을 줄이는 데 활용한다. [RDS PostgreSQL 엔진 업그레이드](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_UpgradeDBInstance.PostgreSQL.html)

## 같이 보면 좋은 학습 포인트

- 샌드박스에서 현재 엔진 버전의 유효한 업그레이드 대상과 자동 적용 대상 표시를 비교한다. [자동 마이너 업그레이드 문서](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_UpgradeDBInstance.PostgreSQL.Minor.html)
- PostgreSQL 메이저 버전별 지원 종료일을 확인해 정기 패치 일정과 메이저 업그레이드 일정을 함께 계획한다. [RDS PostgreSQL 릴리스 일정](https://docs.aws.amazon.com/AmazonRDS/latest/PostgreSQLReleaseNotes/postgresql-release-calendar.html)

## 한 줄 정리

RDS for PostgreSQL이 보안·버그 수정을 포함한 마이너 버전 5종을 지원하므로, 현재 메이저 버전과 리전에 맞춰 검증 및 적용 일정을 잡을 수 있다.
