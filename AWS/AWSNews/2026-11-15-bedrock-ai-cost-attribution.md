# Amazon Bedrock AI 도구의 사용자별 비용 귀속과 사용 통제

## 공식 출처

- [Cost attribution and controls for AI developer and productivity tools on Amazon Bedrock](https://aws.amazon.com/blogs/machine-learning/cost-attribution-and-controls-for-ai-developer-and-productivity-tools-on-amazon-bedrock/)
- 게시일: 2026년 11월 13일

## 핵심 내용

AWS가 Amazon Bedrock을 사용하는 AI 개발 및 생산성 도구의 비용을 사용자, 팀과 비용 센터별로 추적하고 통제하는 운영 패턴을 공개했다. Bedrock은 각 추론 호출의 IAM 주체를 기록하며, 이 정보는 AWS Cost and Usage Report 2.0(CUR 2.0)과 Cost Explorer에서 비용 분석에 활용할 수 있다.

핵심 구성은 다음과 같다.

- 사용자가 AWS IAM Identity Center를 통해 개별적으로 인증한다.
- 기업 IdP의 팀, 부서와 비용 센터 속성을 세션 태그로 전달한다.
- Bedrock이 추론 호출자의 IAM ARN과 연합 사용자 세션 이름을 기록한다.
- CUR 2.0의 `line_item_iam_principal` 열에서 사용자별 사용 비용을 조회한다.
- IAM Principal 태그는 `iamPrincipal/` 접두사가 붙은 비용 할당 태그로 분석할 수 있다.
- Amazon Athena로 사용자, 팀, 비용 센터별 비용을 질의하고 대시보드로 시각화한다.
- CUR보다 더 세밀한 요청 단위 분석은 모델 호출 로그와 요청 메타데이터 태그를 활용한다.
- AWS Budgets, CloudWatch와 Lambda, DynamoDB 기반 할당량 또는 LLM Gateway를 조합해 사용량을 통제한다.

이 글은 Bedrock을 호출하는 특정 AI 도구에 종속된 방식이 아니라 Bedrock 엔드포인트와 IAM 계층에서 신원 및 비용을 연결하는 플랫폼 운영 패턴을 설명한다.

## 왜 중요한가

AI 도구를 조직 전체에 배포하면 청구서에는 총비용이 나타나지만 누가 어떤 업무에 얼마를 썼는지 파악하기 어려울 수 있다. 모든 사용자가 같은 IAM 역할과 고정 세션 이름을 공유하거나 Gateway 역할 하나로 호출하면 사용 비용이 하나의 주체로 합쳐져 팀별 Showback과 Chargeback이 불가능해진다.

사용자 신원을 추론 비용에 연결하면 비용 급증의 원인을 찾고, 팀별 예산을 계획하며, AI 도구가 만든 업무 가치와 지출을 비교할 수 있다. 단순히 비용을 제한하는 데서 끝나지 않고 어떤 모델과 워크로드가 효과적인지 판단하는 FinOps 기반을 만든다는 점이 중요하다.

## 서비스 및 아키텍처 관점

일반적인 비용 귀속 흐름은 다음과 같다.

1. 외부 IdP의 사용자와 그룹을 IAM Identity Center에 동기화한다.
2. 팀과 비용 센터 속성을 세션 태그로 매핑한다.
3. 사용자는 Bedrock 호출 권한이 포함된 Permission Set으로 로그인한다.
4. AI 도구가 사용자의 단기 AWS 자격 증명으로 Bedrock 엔드포인트를 호출한다.
5. Bedrock이 IAM 주체 정보를 사용량 데이터와 연결한다.
6. CUR 2.0이 사용자 신원과 비용 할당 태그를 S3에 전달한다.
7. Athena, Cost Explorer 또는 대시보드가 사용자·팀별 비용을 집계한다.
8. 예산 또는 실시간 통제 계층이 기준을 넘는 사용을 경고하거나 제한한다.

CUR 2.0은 내보내기 간격별 주체와 사용 유형의 집계 비용을 제공하며 요청별 비용 데이터는 아니다. 프롬프트 단위 분석이 필요하면 모델 호출 로그나 요청 메타데이터를 별도로 사용해야 한다.

## 보안 관점

- 장기 액세스 키 대신 IAM Identity Center의 단기 자격 증명을 사용한다.
- Permission Set에는 승인된 모델, 추론 프로필과 프로젝트에 필요한 호출 권한만 부여한다.
- 모든 사용자가 같은 세션 이름을 사용하지 않도록 개인 로그인 흐름을 유지한다.
- 팀과 비용 센터 태그를 사용자가 임의로 변경하지 못하도록 IdP에서 관리한다.
- CloudTrail로 사용자별 Bedrock 호출과 권한 변경을 감사한다.
- 모델 호출 로그에는 프롬프트나 응답의 민감 정보가 포함될 수 있으므로 암호화, 접근 제어와 보관 정책을 적용한다.
- LLM Gateway를 사용할 때 최종 사용자 신원이 비용 데이터까지 유지되도록 설계한다.
- 비용 제한을 위해 적용하는 Deny 정책이 긴급 업무와 운영 시스템에 미칠 영향을 검토한다.

비용 가시성은 보안 통제와도 연결된다. 평소와 다른 사용자별 지출 급증은 자격 증명 오용, 자동화 오류 또는 비정상 호출의 신호가 될 수 있다.

## 운영 관점

통제 방식은 반영 속도와 운영 복잡성이 서로 다르다.

- AWS Budgets: 구축이 간단하지만 결제 데이터 지연 때문에 빠른 과소비를 즉시 막기 어렵다.
- CloudWatch와 Lambda: 모델 또는 추론 프로필 지표를 이용해 수 분 수준의 통제가 가능하다.
- DynamoDB 기반 할당량: 사용자별 토큰 카운터를 관리하고 다음 자격 증명 갱신 시 제한할 수 있다.
- LLM Gateway: 요청마다 차단할 수 있지만 지연과 운영 복잡성이 가장 크다.

운영 시에는 다음 항목을 관리한다.

- 사용자, 팀, 비용 센터별 일간 및 월간 비용
- 입력·출력·캐시 읽기·캐시 쓰기 토큰 비용
- 모델과 추론 프로필별 비용 및 사용량
- 활성 사용자 수와 사용자당 비용 추세
- 비용 할당 태그 누락률과 공통 역할로 집계된 비용
- 예산 경보부터 실제 제한 적용까지의 지연
- 비정상 지출 탐지와 담당자 대응 시간
- AI 도구 사용량과 생산성 또는 업무 성과의 상관관계

보고에는 CUR 2.0을 사용하고, 실시간 제한은 Gateway나 별도 할당량 계층으로 처리하며, AWS Budgets를 최종 안전망으로 두는 계층형 접근이 실용적이다.

## 시험 관점 정리

AWS 시험이나 아키텍처 문제에서는 다음 개념을 연결해서 기억할 수 있다.

- IAM Identity Center는 기업 IdP와 연동해 사용자별 단기 AWS 자격 증명을 제공한다.
- Permission Set은 사용자와 그룹이 AWS 계정에서 사용할 권한을 정의한다.
- CUR 2.0은 상세 사용량과 비용 데이터를 S3로 내보낸다.
- IAM 주체 데이터와 세션 태그를 이용해 Bedrock 비용을 사용자와 조직 단위로 귀속할 수 있다.
- Cost Explorer는 빠른 비용 분석에, Athena는 CUR 데이터의 사용자 지정 질의에 적합하다.
- AWS Budgets는 비용 임계값 알림과 작업을 제공하지만 실시간 차단 도구는 아니다.
- CloudWatch 지표는 사용량 경보와 자동 조치에 사용할 수 있다.
- 최소 권한, 비용 가시성과 사용량 제한은 생성형 AI 거버넌스의 서로 보완적인 요소다.

## 같이 보면 좋은 학습 포인트

- AWS IAM Identity Center와 외부 IdP 연동
- IAM 세션 태그와 ABAC
- CUR 2.0 및 Cost Explorer
- Amazon Athena를 이용한 비용 질의
- Application Inference Profile
- Bedrock 모델 호출 로그
- AWS Budgets와 Budget Actions
- 생성형 AI FinOps와 단위 경제성

## 한 줄 정리

Amazon Bedrock의 IAM 주체 귀속과 CUR 2.0을 활용하면 여러 AI 도구의 비용을 사용자·팀별로 추적하고, 지연 허용 수준에 맞는 통제 계층을 더해 조직 전체의 AI 지출을 관리할 수 있다.
