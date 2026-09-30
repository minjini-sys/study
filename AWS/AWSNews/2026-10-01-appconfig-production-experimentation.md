# AWS AppConfig로 안전하게 프로덕션 A/B 실험 운영하기

- 정리 날짜: 2026-10-01
- 원문 게시일: 2026-09-30
- 공식 출처: [AWS DevOps & Developer Productivity Blog - Running production experiments with AWS AppConfig experimentation](https://aws.amazon.com/blogs/devops/running-production-experiments-with-aws-appconfig-experimentation/)

## 핵심 내용

AWS AppConfig experimentation을 사용하면 별도의 실험 플랫폼을 구축하지 않고 기존 feature flag를 기반으로 A/B 테스트와 다변량 실험을 운영할 수 있다. 실험 가설, 대상 사용자, 대조군과 처리군, 트래픽 비율을 정의하고 실제 프로덕션 트래픽 일부에만 변경 사항을 노출한다.

AWS AppConfig Agent는 애플리케이션 가까이에서 feature flag를 가져와 캐시하고, 동일한 entity ID에는 실험이 진행되는 동안 같은 treatment를 반환한다. 할당 기록은 CloudWatch Logs, Amazon Data Firehose와 Amazon S3로 전송할 수 있으며, 전환율·지연 시간·비용 같은 결과 데이터와 결합해 Athena 또는 기존 데이터 웨어하우스에서 분석한다.

AWS 공식 예제는 프론트엔드의 장바구니 버튼 디자인과 Amazon ECS 백엔드의 캐시 TTL을 실험한다. UI 변경뿐 아니라 추천 알고리즘, AI 모델, 프롬프트와 백엔드 설정도 코드 재배포 없이 비교할 수 있다.

## 왜 중요한가

개발 환경 테스트는 기능이 동작하는지는 보여주지만 실제 사용자의 행동과 프로덕션 부하에 미치는 영향까지 증명하지 못한다. 반대로 변경을 모든 사용자에게 즉시 배포하면 장애의 영향 범위가 커진다.

AppConfig experimentation은 실제 트래픽에서 증거를 얻으면서 노출 비율과 대상 사용자를 제한하는 중간 단계를 제공한다. 조직은 직감 대신 측정 결과로 배포 여부를 결정하고, 부정적인 지표가 나타나면 실험을 중단해 기존 구성으로 되돌릴 수 있다.

## 실험 실행 흐름

1. 검증할 가설과 성공 기준, 악화되면 안 되는 guardrail 지표를 문서화한다.
2. 이미 배포된 AppConfig feature flag를 선택한다.
3. audience rule로 실험 대상의 플랫폼, 지역, 요금제 또는 애플리케이션 버전을 정의한다.
4. 현재 동작인 control과 변경안인 treatment를 구성하고 트래픽 비율을 설정한다.
5. 0% 노출과 treatment override로 기능 및 지표 수집을 먼저 검증한다.
6. 소량의 실제 트래픽부터 점진적으로 노출하며 CloudWatch 경보를 관찰한다.
7. Agent의 treatment 할당 기록과 비즈니스 지표를 entity ID 및 시간 기준으로 결합한다.
8. 승리한 treatment를 feature flag에 배포한 뒤 실험을 중지한다.

## 서비스 및 아키텍처 관점

- **AWS AppConfig**: 실험 정의, audience rule, treatment와 노출 비율을 관리하는 control plane이다.
- **AWS AppConfig Agent**: EC2, ECS, EKS, Lambda 또는 온프레미스 워크로드 가까이에서 구성을 캐시하고 treatment를 전달한다.
- **Amazon CloudWatch**: 애플리케이션 상태와 실험 guardrail 지표를 감시하고 할당 로그를 수집한다.
- **Amazon Data Firehose**: CloudWatch Logs의 treatment 할당 이벤트를 Amazon S3로 전달한다.
- **Amazon S3 및 AWS Glue Data Catalog**: 할당 기록과 결과 데이터를 저장하고 쿼리 가능한 테이블로 관리한다.
- **Amazon Athena 또는 기존 데이터 웨어하우스**: treatment 할당과 전환, 지연 시간, 비용 및 오류 데이터를 조인해 결과를 분석한다.

AppConfig는 실시간 트래픽 분포와 treatment 할당 같은 집계 지표를 제공하지만 실험 결과 자체를 대신 계산하지 않는다. 조직이 기존 분석 플랫폼과 지표 정의를 유지하므로 원시 데이터와 성공 기준에 대한 통제권을 가진다.

## 보안 및 운영 관점

- entity ID는 로그에 그대로 기록되므로 개인정보를 사용하지 않고, 필요하면 양쪽 데이터에서 동일한 방식으로 해시하거나 가명 처리한다.
- AppConfig Agent에는 필요한 구성 읽기 권한만 부여하고 로그 전송 역할도 최소 권한으로 구성한다.
- 실험 시작 전 0% 노출 상태에서 feature flag 평가, 화면 또는 기능 동작, 지표 수집을 확인한다.
- override는 할당 로그를 만들지 않으므로 실제 트래픽을 1% 정도 노출한 뒤 전체 로그 파이프라인도 별도로 검증한다.
- 오류율, p99 지연 시간, 처리량, 비용과 같은 CloudWatch guardrail 경보를 실험 시작 전에 설정한다.
- 경보가 발생하면 자동으로 끝난다고 가정하지 말고 `stop-experiment-run`을 실행하는 운영 절차를 마련한다.
- 실험 중 treatment 동작을 변경하면 결과 해석이 깨지므로 기존 실행을 중지하고 새 실행을 시작한다.
- 승리한 구성을 먼저 배포하고 실험을 중지해야 사용자가 잠시 이전 기본값으로 돌아가는 현상을 피할 수 있다.
- 실행 시간이 과금 기준이므로 필요한 표본을 확보하면 실험을 중지하고 Firehose, 로그 구독과 불필요한 S3 데이터도 정리한다.

## 시험 관점 정리

- AWS AppConfig는 AWS Systems Manager의 기능으로 애플리케이션 구성을 코드 배포와 분리한다.
- feature flag를 사용하면 전체 재배포 없이 기능을 켜고 끄거나 사용자별 값을 전달할 수 있다.
- AppConfig Agent는 로컬 HTTP 엔드포인트를 제공하고 구성을 캐시해 애플리케이션의 API 호출 부담을 줄인다.
- AppConfig 환경 모니터는 구성 배포의 비정상 상태를 감지해 rollback할 수 있지만 실험 실행의 중지는 별도 작업이다.
- CloudWatch는 지표, 로그와 경보를 제공하고 Data Firehose는 스트리밍 데이터를 목적지로 전달한다.
- Athena는 S3의 데이터를 SQL로 분석하며 Glue Data Catalog의 테이블 메타데이터를 사용할 수 있다.
- 프로덕션 실험에서는 점진적 노출, stable assignment, guardrail과 사후 분석이 핵심이다.

## 같이 보면 좋은 학습 포인트

- AWS AppConfig feature flag와 safe deployment 전략
- control, treatment, audience와 statistical power
- 실험의 표본 크기 및 유의 수준 계산
- CloudWatch alarm과 AppConfig environment monitor의 차이
- ECS sidecar 및 Lambda extension 형태의 AppConfig Agent
- CloudWatch Logs subscription filter와 Data Firehose
- S3, Glue, Athena 기반 로그 분석 파이프라인
- post-exposure attribution과 실험 데이터 편향 방지
- AI 프롬프트, 모델과 토큰 비용을 대상으로 한 실험 설계

## 한 줄 정리

AWS AppConfig experimentation은 feature flag, 점진적 노출, 고정된 treatment 할당과 CloudWatch guardrail을 결합해 실제 프로덕션에서 통제된 A/B 실험을 운영하게 해준다.
