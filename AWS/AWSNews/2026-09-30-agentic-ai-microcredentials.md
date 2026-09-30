# AWS Skill Builder, 에이전틱 AI 마이크로크리덴셜 2종 공개

- 정리 날짜: 2026-09-30
- 원문 게시일: 2026-09-29
- 공식 출처: [AWS Training and Certification Blog - Prove your agentic AI skills: Two new Microcredentials now available](https://aws.amazon.com/blogs/training-and-certification/prove-your-agentic-ai-skills-two-new-microcredentials-now-available/)

## 핵심 내용

AWS가 에이전틱 AI 실무 역량을 검증하는 마이크로크리덴셜 2종을 AWS Skill Builder에 공개했다.

- **AWS Securing Agent Identities Demonstrated**: AI 에이전트의 자격 증명, 접근 제어와 감사 체계를 안전하게 구성하는 능력을 평가한다.
- **AWS Databases for Agentic AI Demonstrated**: AI 에이전트가 사용하는 데이터베이스, 메모리와 검색 계층을 구성하고 문제를 해결하는 능력을 평가한다.

두 평가는 구독 없이 무료로 이용할 수 있다. 객관식 시험이 아니라 제한 시간 안에 실제 AWS 환경의 문제를 진단하고 수정하는 실습형 평가이며, 통과하면 Credly 디지털 배지를 받을 수 있다.

## 왜 중요한가

에이전틱 AI는 모델의 응답 품질만으로 완성되지 않는다. 에이전트마다 적절한 권한과 신뢰 경계를 설정하고, 데이터 접근을 통제하며, 실행 기록을 추적할 수 있어야 실제 서비스에 적용할 수 있다.

이번 마이크로크리덴셜은 일반적인 개념 암기보다 실제 구성과 장애 해결 능력을 검증한다. AWS 자격증이 폭넓은 지식을 확인한다면 마이크로크리덴셜은 특정 업무를 직접 수행할 수 있는지를 보여주는 보완 수단이다.

## 마이크로크리덴셜별 학습 범위

### AWS Securing Agent Identities Demonstrated

- 자체 호스팅 에이전트에 워크로드 자격 증명 부여
- 태그 기반 접근 제어 적용
- 범위가 제한된 JWT 권한으로 인바운드 접근 보호
- 호출 주체에 따라 SigV4와 JWT 인증 방식 선택
- 프로덕션 수준의 JWT 검증으로 마이그레이션
- 토큰 작업의 감사 추적 구성
- 관련 서비스: Amazon Bedrock AgentCore, Amazon Cognito, IAM, AWS CloudTrail, Amazon CloudWatch, AWS Systems Manager

### AWS Databases for Agentic AI Demonstrated

- Amazon Aurora PostgreSQL과 Amazon DynamoDB를 에이전트 접근 및 메모리 용도로 구성
- `pgvector`와 Amazon Bedrock Knowledge Bases를 이용한 문서 검색 구현
- MCP 서버를 활용해 에이전트와 데이터베이스 사이의 상호작용 디버깅
- 에이전트가 실행하는 데이터베이스 쿼리 최적화
- 관련 서비스: Amazon Aurora PostgreSQL, Amazon DynamoDB, Amazon Bedrock, AWS Lambda, Amazon CloudWatch, AWS Secrets Manager

## 서비스 및 아키텍처 관점

보안 영역에서는 Amazon Cognito 또는 IAM이 사용자와 워크로드의 신원을 확인하고, Bedrock AgentCore가 에이전트 실행 환경과 연결된다. 외부 사용자는 범위가 제한된 JWT로 접근하고 AWS 서비스 간 호출은 SigV4로 서명하는 식으로 호출 주체에 맞는 인증 방식을 선택해야 한다.

데이터 영역에서는 Aurora PostgreSQL과 `pgvector`가 관계형 데이터 및 벡터 검색을 담당할 수 있고, DynamoDB는 세션 상태나 장기 메모리를 낮은 지연 시간으로 저장할 수 있다. Bedrock Knowledge Bases는 문서 검색과 모델 응답 생성을 연결하며 Lambda는 데이터 처리나 도구 호출 로직을 실행한다.

두 영역 모두 CloudWatch로 로그와 지표를 관찰하고 CloudTrail로 API 활동을 감사해야 한다. 데이터베이스 자격 증명은 Secrets Manager에 저장하고 애플리케이션 코드에 직접 포함하지 않는다.

## 보안 및 운영 관점

- 에이전트마다 전용 워크로드 자격 증명을 사용하고 사람의 장기 자격 증명을 공유하지 않는다.
- IAM 정책은 최소 권한으로 구성하고 태그 기반 접근 제어를 사용할 때 태그 변경 권한도 함께 제한한다.
- JWT의 발급자, 대상, 서명, 만료 시간과 허용 범위를 모두 검증한다.
- 토큰 발급과 교환, 권한 거부와 비정상 API 호출을 CloudTrail 및 CloudWatch에서 추적한다.
- Secrets Manager를 사용해 데이터베이스 비밀 정보를 중앙 관리하고 주기적으로 교체한다.
- 에이전트의 쿼리 패턴, 지연 시간, 오류율과 사용량을 관찰해 비용 및 성능 문제를 조기에 찾는다.
- 프롬프트 입력만 신뢰하지 말고 도구 호출과 데이터 접근 계층에서 별도의 권한 검사를 수행한다.

## 시험 및 학습 관점 정리

- **IAM**은 AWS 리소스 접근을 제어하며 사용자, 역할과 정책을 기반으로 최소 권한을 구현한다.
- **Amazon Cognito**는 애플리케이션 사용자의 가입, 로그인과 토큰 발급을 지원한다.
- **SigV4**는 AWS API 요청의 신원과 무결성을 확인하는 서명 방식이다.
- **JWT**는 클레임을 담은 토큰이며 서명뿐 아니라 발급자, 대상과 만료 조건을 검증해야 한다.
- **CloudTrail**은 AWS 계정의 API 활동을 기록하고 **CloudWatch**는 로그, 지표와 경보를 제공한다.
- **Aurora PostgreSQL**은 PostgreSQL 호환 관계형 데이터베이스이고 `pgvector` 확장으로 벡터 데이터를 다룰 수 있다.
- **DynamoDB**는 서버리스 NoSQL 데이터베이스로 에이전트의 상태와 메모리 저장에 활용할 수 있다.
- **Bedrock Knowledge Bases**는 검색 증강 생성에 필요한 데이터 검색 흐름을 관리한다.
- 마이크로크리덴셜은 AWS Certification을 대체하기보다 특정 기술의 실습 역량을 추가로 증명한다.

## 같이 보면 좋은 학습 포인트

- Bedrock AgentCore의 Identity 및 Runtime 구성
- IAM 역할 신뢰 정책과 임시 자격 증명
- OAuth 2.0, OpenID Connect, JWT와 SigV4의 차이
- 속성 기반 접근 제어(ABAC)와 태그 거버넌스
- CloudTrail 이벤트 기록과 CloudWatch 경보 설계
- Aurora PostgreSQL `pgvector` 인덱스와 유사도 검색
- DynamoDB 파티션 키 설계와 일관성 모델
- RAG와 Bedrock Knowledge Bases의 데이터 수집 흐름
- MCP 서버의 권한 경계와 비밀 정보 관리

## 한 줄 정리

AWS의 새로운 에이전틱 AI 마이크로크리덴셜은 에이전트의 신원 보안과 데이터 계층을 실제 AWS 환경에서 구성하고 문제를 해결하는 능력을 무료 실습 평가로 검증한다.
