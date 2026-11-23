# AWS, API 및 AI 엔드포인트 보안 기능 정식 출시

- 정리 날짜: 2026-11-24
- 공식 발표일: 2026-11-23
- 공식 출처: [Introducing API and AI security on AWS, from discovery to protection](https://aws.amazon.com/blogs/security/introducing-api-and-ai-security-on-aws-from-discovery-to-protection/)
- 공식 문서: [AWS WAF AI Security managed rule group](https://docs.aws.amazon.com/waf/latest/developerguide/aws-managed-rule-groups-ai-security.html)

## 핵심 내용

AWS가 API와 AI 엔드포인트를 발견하고 보호하기 위한 세 가지 기능을 정식 출시했다.

- **API Portal**: Amazon API Gateway의 새 기능으로 API와 AI 엔드포인트를 인벤토리화한다.
- **API Security**: 일반 API 공격과 비정상 행동을 탐지·차단하는 AWS WAF 관리형 규칙 그룹이다.
- **AI Security**: LLM 추론, Model Context Protocol(MCP), Agent2Agent(A2A) 트래픽을 인식하고 프로토콜 및 콘텐츠를 검사하는 AWS WAF 관리형 규칙 그룹이다.

API Portal은 Amazon CloudFront, Application Load Balancer(ALB), Amazon API Gateway의 구성과 실제 트래픽을 분석해 REST API뿐 아니라 LLM, MCP, A2A 엔드포인트도 찾아낸다. 단일 계정 또는 AWS Organizations 조직 단위로 검색을 활성화할 수 있어 문서화되지 않은 섀도 API와 보호되지 않은 엔드포인트를 확인하는 데 도움을 준다.

API Security는 객체 수준 권한 부재 악용, 변조된 토큰, 자격 증명 대입, 존재하지 않는 경로 탐색과 파라미터 퍼징 등을 탐지한다. 단일 요청을 검사하는 결정적 규칙과 정상 트래픽 기준선을 학습하는 행동 기반 탐지를 함께 사용한다.

AI Security는 요청을 LLM, MCP, A2A 또는 미인식 트래픽으로 분류하고, MCP와 A2A 메시지가 선언한 프로토콜 규격을 따르는지 검사한다. 선택적으로 Amazon Bedrock Guardrails를 호출해 프롬프트 공격, 유해 콘텐츠 및 응답의 민감 정보를 평가한다.

## 왜 중요한가

기존 웹 애플리케이션의 API는 주로 서비스나 사람이 사용하는 앱에서 호출됐다. 이제 AI 에이전트가 사용자를 대신해 API를 선택하고 여러 호출을 연결하면서 정상적인 자동화와 공격성 자동화의 행동이 비슷해졌다. 브라우저 지문이나 CAPTCHA만으로는 서버 간 요청과 에이전트 요청을 충분히 구분하기 어렵다.

또한 조직이 보유한 모든 API를 알지 못하면 보안 규칙을 적용할 대상도 결정할 수 없다. API Portal은 먼저 엔드포인트를 발견하고, API Security와 AI Security는 같은 AWS WAF 보호 팩(web ACL)에서 각 트래픽의 특성에 맞게 보호한다. 발견과 보호를 연결한다는 점이 핵심이다.

AI Security는 일반적인 웹 공격 검사에 더해 MCP·A2A 프로토콜 위반과 프롬프트 인젝션, 시스템 프롬프트 탈취, 자격 증명이나 개인정보 노출 같은 AI 고유 위험을 다룬다. 별도의 프록시 서비스를 배포하지 않고 기존 AWS WAF 계층에서 시작할 수 있다는 운영상 이점도 있다.

## 서비스·아키텍처 관점

권장 흐름은 다음과 같이 구성할 수 있다.

1. API Portal을 계정 또는 AWS Organizations 범위에서 활성화한다.
2. CloudFront, ALB, API Gateway의 구성과 트래픽을 기반으로 API 인벤토리를 만든다.
3. 중요한 엔드포인트를 관리 대상 API로 승격하고 소유자와 문서를 연결한다.
4. 엔드포인트 앞의 AWS WAF web ACL에 API Security와 AI Security 관리형 규칙을 추가한다.
5. 탐지 결과가 추가한 AWS WAF 라벨을 후속 사용자 정의 규칙, 속도 제한 및 경보에 활용한다.

세 기능은 CloudFront, ALB, API Gateway REST API를 중심으로 동작한다. 인터넷 공개 API 앞에 CloudFront를 두면 엣지에서 요청을 검사하고 원본에 도달하기 전에 공격을 완화할 수 있다. CloudFront VPC 오리진을 사용하면 AWS 내부 원본을 직접 인터넷에 공개하지 않는 구조도 고려할 수 있다.

API Security와 AI Security의 탐지는 요청에 라벨로 추가된다. 이후 규칙에서 Bot Control, Fraud Control, IP 평판 같은 다른 신호와 라벨을 조합하거나 특정 라벨이 반복되는 IP에만 속도 제한을 적용할 수 있다. 한 가지 약한 신호만으로 차단하기보다 여러 신호를 결합하는 계층형 정책을 만들 수 있다.

AI Security는 지원되는 작업의 메시지 콘텐츠를 Bedrock Guardrails로 보낼 수 있다. CloudFront를 보호할 때는 엣지 위치가 가까운 지원 리전으로 콘텐츠를 전송할 수 있고 사용자가 리전을 선택할 수 없으므로, 데이터 레지던시 요구 사항이 있는 아키텍처는 이 처리 경로를 반드시 검토해야 한다.

## 보안 관점

API Portal로 인벤토리를 확보한 뒤 각 엔드포인트에 인증, 인가, WAF 및 로깅이 적용됐는지 확인해야 한다. API Portal 자체 권한은 `apiportal:` IAM 작업으로 제어하고, 조직 검색은 관리 계정 대신 위임 관리자 계정에서 운영하는 것이 좋다.

AI Security의 Bedrock Guardrails 콘텐츠 검사를 사용하려면 AWS WAF가 수임할 IAM 서비스 역할을 제공해야 한다. 이 역할에는 필요한 Guardrails 호출 권한만 부여하고 신뢰 정책의 주체를 제한해야 한다. Guardrails 호출 실패 시 기본적으로 요청이 통과할 수 있으므로, 보안 요구 수준에 따라 `Guardrails_CallFailed` 규칙을 차단으로 바꾸는 fail-closed 정책을 검토해야 한다. 다만 스로틀링도 차단으로 이어질 수 있어 가용성과 보안의 균형이 필요하다.

응답 검사는 활성화해야 동작하며, AWS WAF가 검사하는 응답 본문 크기를 넘는 콘텐츠는 평가되지 않는다. 따라서 민감 정보 유출 방지를 AI Security 하나에만 의존하지 말고 최소 권한, 데이터 분류, 애플리케이션 출력 검증과 Bedrock Guardrails의 애플리케이션 계층 제어를 함께 사용해야 한다.

MCP 서버의 인증과 도구별 권한 검사는 여전히 애플리케이션 책임이다. AI Security가 프로토콜과 콘텐츠를 검사하더라도 JWT 검증, OAuth 토큰 범위, 도구 호출 인가 및 사용자 의도 확인을 대신하지 않는다.

## 운영 관점

- 관리형 규칙을 처음 적용할 때 규칙 동작을 `Count`로 재정의해 실제 트래픽의 오탐을 확인한 후 단계적으로 차단한다.
- 행동 기반 API Security 탐지가 정상 기준선을 학습할 시간을 확보하고 학습 기간에는 지표를 관찰한다.
- AWS WAF 로그와 CloudWatch 지표에서 API Security·AI Security 라벨별 발생량과 차단 결과를 모니터링한다.
- `guardrails:not_assessed`, `guardrails:call_failed`, 스로틀링 관련 라벨을 별도로 경보한다.
- AI Security 적용 범위를 실제 LLM·MCP·A2A 경로로 제한해 비용과 불필요한 검사를 줄인다.
- API Portal에서 새로 발견된 엔드포인트의 소유자, 데이터 등급, 인증 방식과 web ACL 연결 여부를 정기 검토한다.
- 응답 검사 활성화 여부와 본문 크기 제한을 테스트하고, 큰 응답에서 민감 정보가 누락되지 않는지 확인한다.
- 관리형 규칙 버전과 변경 로그를 추적하고 프로덕션 적용 전에 회귀 테스트한다.

## 시험 관점 정리

AWS 자격시험에서는 다음 연결 관계를 기억하면 좋다.

- **API Portal**: 조직의 API와 AI 엔드포인트를 발견하고 인벤토리화
- **AWS WAF API Security**: 일반 API 권한 악용, 인증 공격 및 정찰 행동을 탐지
- **AWS WAF AI Security**: LLM·MCP·A2A 트래픽 인식, 프로토콜 검사 및 AI 콘텐츠 보호
- **Amazon Bedrock Guardrails**: 프롬프트 공격, 유해 콘텐츠와 민감 정보 평가
- **AWS WAF 라벨**: 관리형 규칙의 탐지 결과를 후속 규칙, 속도 제한 및 지표에서 재사용
- **Count 모드**: 차단 전에 실제 트래픽에서 오탐과 영향 범위를 평가
- **CloudFront**: 엣지에서 AWS WAF를 적용해 원본 도달 전 요청을 검사

섀도 API를 먼저 찾고 여러 계정의 API 보안 상태를 파악해야 한다면 API Portal이 출발점이다. MCP나 A2A 엔드포인트의 프로토콜 위반과 프롬프트 공격을 AWS 엣지·진입 계층에서 검사하려면 AI Security가 적합한 후보가 된다.

## 같이 보면 좋은 학습 포인트

1. AWS WAF web ACL, 관리형 규칙 그룹, 우선순위와 라벨의 동작 방식
2. OWASP API Security Top 10과 객체 수준 권한 부재(BOLA)
3. MCP와 A2A의 요청 구조 및 인증·인가 책임
4. Amazon Bedrock Guardrails의 입력·출력 평가 기능
5. CloudFront, ALB, API Gateway에 AWS WAF를 연결하는 방법
6. AWS Organizations 위임 관리자와 중앙 보안 운영 모델
7. WAF 로그, CloudWatch 지표와 SIEM을 이용한 탐지 대응

## 한 줄 정리

AWS의 API Portal·API Security·AI Security는 조직의 일반 API와 LLM·MCP·A2A 엔드포인트를 먼저 발견하고, AWS WAF와 Bedrock Guardrails로 프로토콜 공격과 콘텐츠 위험까지 진입 계층에서 보호한다.
