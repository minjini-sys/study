# Amazon Quick에 항상 실행되는 에이전트와 엔터프라이즈 제어 기능 추가

날짜: 2026-09-15  
출처: [AWS What's New - Amazon Quick adds always-on agents, a sharper feed, and enterprise controls](https://aws.amazon.com/about-aws/whats-new/2026/09/amazon-quick-always-on-agents-sharper-feed-enterprise-controls/)

## 오늘의 AWS 소식

AWS가 Amazon Quick에 새로운 생산성 및 엔터프라이즈 제어 기능을 추가했다.

Amazon Quick은 업무용 AI 어시스턴트로, 리서치, 비즈니스 인사이트, 자동화, 노코드 앱 생성을 지원하는 서비스이다.

이번 업데이트의 핵심은 에이전트가 사용자가 자리를 비운 동안에도 클라우드에서 계속 동작하고, 조직 단위 관리와 검증 기능이 강화되었다는 점이다.

## 핵심 내용

이번 발표에서 추가된 주요 기능은 다음과 같다.

- Scheduled tasks와 monitoring agents가 클라우드에서 계속 실행
- 노트북이 닫혀 있어도 결과가 activity feed로 전달
- Activity feed에 상위 필터와 개선된 catch-up view 추가
- Daily briefing이 하루 세 번 새로고침
- 최대 7일치 feed 데이터 검색 지원
- 모바일 디바이스 관리(MDM) 지원
- 사용자별 권한 설정 지원
- Microsoft Purview DLP 통합
- 파일 공유, 에이전트 및 스킬 게시 기능
- 공식 스킬 카탈로그 탐색 지원
- Quick spaces와 Quick apps의 데스크톱 네이티브 실행
- 로컬 폴더를 검색 가능한 space로 전환 가능
- 답변 검증을 위한 inline citation 제공
- Professional 및 Enterprise 요금제의 agent hour 증가

## 왜 중요한가

AI 어시스턴트가 실제 업무에 들어가려면 단순 질문 답변만으로는 부족하다.

기업 환경에서는 다음 조건이 필요하다.

- 사용자가 자리를 비워도 자동화가 계속 실행될 것
- 결과를 추적할 수 있을 것
- 답변의 출처를 검증할 수 있을 것
- 조직 정책에 맞게 권한과 데이터 유출 방지를 적용할 수 있을 것
- 팀 단위로 에이전트와 스킬을 재사용할 수 있을 것

이번 Amazon Quick 업데이트는 이런 엔터프라이즈 요구 사항을 직접 겨냥한다.

## 아키텍처 관점

Amazon Quick은 전통적인 AWS 인프라 서비스라기보다 업무용 AI 애플리케이션에 가깝다.

하지만 아키텍처 관점에서는 다음 흐름을 보여준다.

- AI 에이전트가 단발성 응답에서 지속 실행 작업으로 확장
- 로컬 데이터와 클라우드 기반 에이전트가 결합
- 조직의 보안 정책과 AI 워크플로가 통합
- 답변 신뢰성을 위해 citation과 데이터 출처 확인이 중요해짐

즉, 생성형 AI 서비스는 점점 더 "질문하면 답하는 도구"에서 "업무를 지속적으로 처리하는 운영 주체"로 이동하고 있다.

## 보안과 거버넌스

이번 업데이트에서 특히 중요한 부분은 엔터프라이즈 제어 기능이다.

관리자는 모바일 디바이스 관리, 사용자별 권한, Microsoft Purview DLP 통합을 통해 조직 정책을 적용할 수 있다.

AI 도구가 사내 파일과 업무 데이터에 접근할수록 다음 통제가 중요해진다.

- 어떤 데이터에 접근할 수 있는지
- 누가 에이전트를 만들고 배포할 수 있는지
- 민감 정보가 외부로 나가지 않는지
- 답변의 근거를 확인할 수 있는지
- 자동화 결과가 어디에 저장되는지

## 시험 관점에서 정리

AWS 자격증 관점에서는 Amazon Quick 자체보다 AI 도구의 운영 요구 사항을 이해하는 것이 중요하다.

정리하면 다음과 같다.

- 엔터프라이즈 AI는 보안, 권한, 감사, DLP가 핵심이다.
- 에이전트 기반 자동화는 지속 실행과 모니터링이 필요하다.
- 답변 검증을 위해 citation과 데이터 출처 확인이 중요하다.
- 조직 내 AI 도입은 기술 기능뿐 아니라 거버넌스 체계를 함께 설계해야 한다.

## 같이 보면 좋은 학습 포인트

AWS Skill Builder에서는 다음 주제를 함께 보면 좋다.

- Generative AI on AWS
- Responsible AI
- AWS security best practices
- IAM access control
- Data governance
- AI agent workflows

참고: [AWS Skill Builder](https://skillbuilder.aws/)

## 한 줄 정리

<mark>Amazon Quick은 항상 실행되는 에이전트, 강화된 feed, DLP와 권한 제어를 추가해 기업 환경에서 AI 업무 자동화를 더 안전하게 운영할 수 있도록 확장되었다.</mark>
