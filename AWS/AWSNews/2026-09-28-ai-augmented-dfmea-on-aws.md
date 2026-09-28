# AWS에서 멀티 에이전트로 자동차 DFMEA 분석하기

- 정리 날짜: 2026-09-28
- 원문 게시일: 2026-09-27
- 공식 출처: [AWS for Industries Blog - Building AI-augmented B-pillar DFMEA on AWS](https://aws.amazon.com/blogs/industries/building-ai-augmented-b-pillar-dfmea-on-aws-architecture-multi-agent-orchestration-and-implementation/)

## 핵심 내용

AWS가 자동차 B-pillar 설계의 고장 형태 및 영향 분석(DFMEA)을 AI로 보조하는 참조 아키텍처를 공개했다. 이 구조는 Amazon Bedrock AgentCore와 Strands Agents SDK 기반 멀티 에이전트, Amazon Neptune 지식 그래프, 서버리스 문서 처리 파이프라인과 사람의 승인 단계를 결합한다.

핵심은 파운데이션 모델이 과거 문서의 표현을 단순히 검색하거나 흉내 내는 데 그치지 않도록 공학 온톨로지를 사용하는 것이다. 온톨로지는 부품, 재료, 제조 공정, 기능, 고장 메커니즘과 영향을 관계로 연결하며 에이전트는 이 그래프를 따라 고장의 인과 관계를 찾는다.

## 왜 중요한가

DFMEA는 설계 단계에서 발생 가능한 고장과 영향을 찾아 위험을 줄이는 활동이다. 하지만 복잡한 차량 부품은 인터페이스와 재료·공정 조합이 많아 사람의 경험과 기억만으로 모든 가능성을 검토하기 어렵다. AWS 게시물에 따르면 수작업 분석은 잠재 고장 형태의 40~60%를 놓칠 수 있다.

지식 그래프와 전문 에이전트를 함께 사용하면 여러 관점의 분석을 병렬로 수행하고 각 결과가 어떤 온톨로지 경로와 근거 문서에서 나왔는지 추적할 수 있다. AI가 최종 결정을 대신하는 것이 아니라 반복 탐색을 자동화하고 중요한 판단은 전문가가 승인하는 구조다.

## 서비스 및 아키텍처 관점

전체 흐름은 다음과 같이 구성된다.

1. Amazon S3와 CloudFront가 React 화면을 제공하고 Amazon Cognito가 사용자를 인증한다.
2. API Gateway와 Lambda가 요청을 받아 AWS Step Functions 분석 워크플로를 시작한다.
3. 설계 문서는 S3에 업로드되고 EventBridge와 별도 Step Functions 흐름이 Amazon Textract 비동기 OCR을 실행한다.
4. 빠른 통계 모델이 위험도와 이상 징후를 먼저 선별한 뒤 전문 AI 에이전트가 상세 분석을 수행한다.
5. AgentCore Runtime의 전문 에이전트가 MCP 도구를 통해 Neptune, Bedrock Knowledge Bases, DynamoDB와 S3의 근거를 조회한다.
6. 분석 에이전트가 전문 에이전트의 결과를 통합하고 최종 DFMEA 보고서를 생성한다.
7. 주요 단계마다 사람의 승인을 기다리고 SNS가 검토자에게 알림을 보낸다.

비용이 낮고 결정적인 작업은 기존 머신러닝으로 먼저 처리하고 복잡한 추론에만 LLM을 사용하는 `cheap-before-expensive` 원칙도 적용됐다.

## 온톨로지와 버전 관리

Neptune은 재료, 공정, 기능, 고장 메커니즘과 영향의 관계를 그래프로 저장한다. 분석 세션이 시작되면 특정 온톨로지 버전에 고정되므로 분석 중 지식 그래프가 갱신돼도 결과의 일관성과 재현성을 유지할 수 있다.

새로운 지식과 관계는 전문가 승인 후 반영되며, 버전이 지정된 스냅샷은 S3에 저장된다. 각 변경과 승인 기록을 남겨 어떤 지식에 근거해 분석했는지 감사할 수 있다.

## 보안 및 운영 관점

- Amazon Cognito, IAM과 AgentCore의 기계 간 인증을 분리하고 역할마다 최소 권한을 적용한다.
- AWS KMS로 문서, 분석 결과, 지식 그래프 관련 데이터를 암호화한다.
- 업로드에는 S3 사전 서명 URL을 사용하고 AWS WAF로 외부 요청을 필터링한다.
- 에이전트가 사용하는 MCP 도구와 데이터 소스를 허용 목록으로 제한한다.
- 낮은 신뢰도 결과, 에이전트 간 모순, 온톨로지 변경과 최종 보고서에는 사람의 승인 단계를 둔다.
- 결정, 시각, 근거와 수동 변경 내용을 변경 불가능한 감사 패키지로 보존한다.
- WebSocket 진행 상황과 Step Functions 실행 이력을 이용해 장시간 분석 작업을 관찰한다.

## 시험 관점 정리

- AWS Step Functions는 여러 서버리스 작업과 승인 단계를 상태 머신으로 조정한다.
- Amazon Neptune은 관계를 그래프로 표현하고 탐색하는 관리형 그래프 데이터베이스다.
- Amazon Bedrock Knowledge Bases는 생성형 AI 답변을 기업 데이터에 근거하도록 돕는다.
- Amazon Textract는 문서에서 텍스트와 구조화된 정보를 추출한다.
- Amazon SNS는 비동기 알림을 전달하며 EventBridge는 이벤트 기반 서비스 연결에 사용된다.
- 중요 산업의 AI 시스템에서는 사람의 검토, 추적 가능한 근거, 버전 고정과 감사 로그가 중요하다.

## 같이 보면 좋은 학습 포인트

- DFMEA와 Risk Priority Number(RPN)
- Amazon Neptune과 SPARQL 그래프 질의
- Amazon Bedrock AgentCore Runtime, Memory와 Gateway
- Strands Agents의 Agents as Tools 패턴
- Step Functions의 `waitForTaskToken` 승인 패턴
- RAG와 지식 그래프를 결합한 GraphRAG
- 생성형 AI 시스템의 Human-in-the-Loop 설계

## 한 줄 정리

AWS의 AI 보조 DFMEA 아키텍처는 공학 온톨로지와 전문 멀티 에이전트, 사람의 승인 단계를 결합해 자동차 설계 위험 분석의 누락을 줄이고 결과의 근거와 감사 가능성을 높인다.
