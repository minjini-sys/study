# Lake Formation과 Trusted Identity Propagation으로 사용자별 AI 데이터 접근 제어

- 정리 날짜: 2026-10-08
- 원문 게시일: 2026-10-06
- 공식 출처: [AWS Security Blog - Identity-aware AI data agents with AWS Lake Formation and Trusted Identity Propagation](https://aws.amazon.com/blogs/security/identity-aware-ai-data-agents-with-aws-lake-formation-and-trusted-identity-propagation/)

## 핵심 내용

AWS가 Amazon Bedrock AgentCore, AWS IAM Identity Center와 AWS Lake Formation을 결합해 AI 데이터 에이전트가 실제 사용자의 권한으로 lakehouse를 조회하는 아키텍처를 공개했다.

일반적인 데이터 에이전트는 도구의 IAM 역할로 Athena 쿼리를 실행한다. 이 경우 Lake Formation은 질문한 사용자가 아니라 도구 역할만 보게 되어 모든 사용자에게 같은 데이터 권한을 적용하거나 애플리케이션 코드에서 권한 로직을 다시 구현해야 한다.

새 패턴은 Trusted Identity Propagation(TIP)으로 사용자의 identity를 UI, AgentCore Runtime, AgentCore Gateway와 Lambda를 거쳐 데이터 계층까지 전달한다. Lake Formation은 기존 사용자 또는 그룹 grant를 그대로 평가하고, CloudTrail은 실제로 데이터에 접근한 사람을 `onBehalfOf` 정보로 기록한다.

## 왜 중요한가

자연어 데이터 에이전트는 사용자가 SQL을 몰라도 분석할 수 있게 하지만, 에이전트의 광범위한 서비스 역할을 그대로 사용하면 권한이 낮은 사용자도 민감한 테이블이나 열을 조회할 위험이 있다. 반대로 애플리케이션에서 사용자별 필터를 직접 구현하면 데이터 거버넌스가 여러 코드베이스로 분산된다.

Trusted Identity Propagation은 인증된 사용자의 identity를 데이터 계층까지 보존한다. 권한 결정은 에이전트나 foundation model이 아니라 Lake Formation에서 수행하므로 기존 행, 열과 테이블 수준 정책을 AI 워크로드에도 재사용할 수 있다.

또한 사용자의 identity token을 prompt나 tool parameter에 넣지 않고 HTTP transport에서만 전달한다. 모델은 token 내용을 볼 수 없으며 Lambda가 서버 측에서 단일 invocation 동안만 token을 교환한다.

## 두 종류의 토큰

- **access token**: AgentCore Runtime과 Gateway 등 각 trust boundary에서 요청을 인증한다.
- **ID token**: 실제 사용자의 identity를 나타내며 Lambda에서 IAM Identity Center identity context로 교환된다.

access token은 표준 `Authorization` header에 넣는다. ID token은 `X-Amzn-Bedrock-AgentCore-Runtime-Custom-IdToken` 같은 사용자 정의 header로 전달한다. HTTP body에는 사용자의 자연어 질문만 포함하고 token은 모델의 context, memory, trace 또는 tool schema에 넣지 않는다.

## 요청 흐름

1. 사용자가 Amazon Cognito 또는 다른 OIDC identity provider에 로그인해 access token과 ID token을 받는다.
2. UI가 AgentCore Runtime을 HTTPS로 호출하며 access token은 `Authorization`, ID token은 사용자 정의 header에 넣는다.
3. Runtime이 JWT authorizer로 access token을 검증하고 허용 목록에 등록된 header만 agent container로 전달한다.
4. agent는 두 header를 HTTP transport에 유지한 채 MCP 연결로 AgentCore Gateway를 호출한다.
5. Gateway가 access token을 검증하고 허용된 ID token header를 Lambda target의 client context로 전달한다.
6. Lambda가 ID token을 검증하고 IAM Identity Center의 `CreateTokenWithIAM` 흐름으로 identity context를 만든다.
7. Lambda가 `ProvidedContexts`에 identity context를 담아 TIP role을 assume한다.
8. 해당 short-lived session으로 Athena query를 실행한다.
9. Lake Formation이 TIP role이 아니라 전파된 실제 사용자의 grant를 평가해 허용된 행과 열만 반환한다.
10. CloudTrail이 `AssumeRole` 이벤트에 실제 사용자를 `onBehalfOf`로 기록한다.

## 서비스 및 아키텍처 관점

- **OIDC IdP 및 Amazon Cognito**: 사용자를 인증하고 ID token과 access token을 발급한다.
- **AWS IAM Identity Center**: trusted token issuer와 OAuth application을 통해 외부 identity를 AWS identity context로 교환한다.
- **Amazon Bedrock AgentCore Runtime**: agent container를 실행하고 허용된 request header를 코드에 전달한다.
- **Amazon Bedrock AgentCore Gateway**: MCP tool endpoint를 제공하고 지정된 header를 Lambda target으로 전달한다.
- **AWS Lambda**: token 검증과 서버 측 교환, TIP role assume 및 Athena query 실행을 담당한다.
- **AWS Lake Formation**: 실제 사용자 또는 그룹의 데이터 grant와 행 및 열 필터를 평가한다.
- **Amazon Athena**: 사용자 identity가 포함된 임시 session으로 lakehouse query를 실행한다.
- **AWS CloudTrail**: 누가 누구를 대신해 role을 assume하고 데이터에 접근했는지 감사 기록을 남긴다.

TIP role 자체에는 Lake Formation 데이터 grant를 주지 않는다. 이 역할은 Athena, Glue catalog, `lakeformation:GetDataAccess`와 Athena 결과용 S3 bucket 등 서비스 호출에 필요한 IAM 권한만 가진다. 데이터 접근 여부의 주체는 role이 아니라 전파된 사용자 identity다.

## 보안 및 운영 관점

- ID token을 prompt, tool argument, model memory 또는 trace에 넣지 않고 HTTP header로만 전달한다.
- AgentCore Runtime의 `requestHeaderAllowlist`에는 필요한 표준 및 사용자 정의 header만 등록한다.
- Gateway target의 `metadataConfiguration.allowedRequestHeaders`도 동일하게 최소화한다.
- Lambda는 token의 signature, issuer, audience와 expiration을 서버 측에서 검증한다.
- identity context는 하나의 Lambda invocation 안에서 생성, 사용하고 외부로 반환하지 않는다.
- TIP role에는 Lake Formation 데이터 grant를 부여하지 않아 권한 우회를 막는다.
- Lake Formation의 사용자 및 그룹 grant를 권한 원본으로 유지하고 애플리케이션 코드에 중복 권한 로직을 만들지 않는다.
- Athena 결과 bucket은 암호화하고 사용자별 query 결과 접근 및 보존 정책을 별도로 검토한다.
- CloudTrail의 `onBehalfOf`를 이용해 사용자별 query 감사 증거를 검증한다.
- 권한이 있는 사용자와 없는 사용자로 같은 질문을 실행해 결과가 다르게 적용되는지 end-to-end 테스트한다.
- token이나 identity context가 오류 메시지와 애플리케이션 로그에 출력되지 않도록 redaction을 적용한다.

## 시험 관점 정리

- IAM Identity Center는 workforce identity의 AWS 계정 및 애플리케이션 접근을 중앙 관리한다.
- Trusted Identity Propagation은 사용자의 identity를 지원되는 AWS 서비스 사이에 전달한다.
- Lake Formation은 Glue Data Catalog 리소스와 데이터 lake에 세분화된 접근 제어를 제공한다.
- Athena는 S3 데이터를 SQL로 조회하며 Lake Formation 권한과 통합할 수 있다.
- OAuth 2.0 access token은 API 접근 권한을 나타내고 OpenID Connect ID token은 인증된 사용자 정보를 나타낸다.
- IAM role의 권한과 Lake Formation 데이터 grant는 서로 다른 계층이며 둘 다 올바르게 구성해야 한다.
- short-lived credential과 server-side token exchange는 장기 credential 노출 위험을 줄인다.
- CloudTrail은 role session에 전파된 identity 정보를 기록해 사람 단위 감사 추적을 지원한다.

## 같이 보면 좋은 학습 포인트

- OAuth 2.0 JWT Bearer grant와 OpenID Connect
- IAM Identity Center trusted token issuer
- STS `AssumeRole`의 `ProvidedContexts`
- Lake Formation table, column 및 row-level grant
- Bedrock AgentCore Runtime custom header allow list
- AgentCore Gateway MCP target와 header propagation
- Athena workgroup 및 query result bucket 보안
- CloudTrail `onBehalfOf` 감사 기록
- confused deputy 방지와 delegated authorization
- agent service role과 사용자 identity 권한의 분리

## 한 줄 정리

Trusted Identity Propagation을 사용하면 AI 에이전트가 광범위한 도구 역할이 아니라 실제 사용자의 Lake Formation 권한으로 데이터를 조회하고, token은 모델에 노출하지 않은 채 사용자 단위 감사 기록을 남길 수 있다.
