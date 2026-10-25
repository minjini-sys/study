# Bedrock Web Search의 도메인 필터와 사용자 위치 제어

- 정리 날짜: 2026-10-26
- 원문 게시일: 2026-10-23
- 공식 출처: [AWS What's New - Web Search on AWS Bedrock now supports domain filtering and user location controls](https://aws.amazon.com/about-aws/whats-new/2026/10/bedrock-web-search-domain-user-location/)

## 핵심 내용

Amazon Bedrock의 Web Search가 model response를 grounding할 검색 결과를 더 정밀하게 제어하는 두 가지 기능을 지원한다.

1. **Domain filtering**: 검색 결과에 포함할 domain 또는 제외할 domain을 지정한다.
2. **User location**: 사용자의 지역 정보를 선택적으로 전달해 지역과 관련된 검색 결과를 받는다.

두 기능은 기존 OpenAI Responses API request의 `web_search` tool에서 직접 설정한다. domain filter에는 최대 100개의 `allowed_domains` 또는 최대 100개의 `blocked_domains`를 지정할 수 있다.

`allowed_domains`는 승인한 source만 검색 결과에 포함하는 allow list 방식이고, `blocked_domains`는 원하지 않는 source를 제외하는 deny list 방식이다. user location은 지역별 service, 규정, 매장, 행사와 availability처럼 장소에 따라 답이 달라지는 질문에 사용할 수 있다.

## 왜 중요한가

web-grounded AI는 최신 정보를 사용할 수 있지만 검색 engine이 찾은 모든 source가 같은 신뢰도를 갖는 것은 아니다. 공식 documentation만 사용해야 하는 기술 지원 agent가 community 글이나 오래된 복제 문서를 인용하면 정확성과 감사 가능성이 떨어질 수 있다.

domain allow list를 사용하면 정부, 기업 공식 사이트 또는 검증된 knowledge publisher처럼 조직이 승인한 source로 retrieval 범위를 제한할 수 있다. 반대로 특정 저품질 source, competitor domain 또는 policy상 사용하면 안 되는 site만 제외하려면 block list가 더 유연하다.

user location은 같은 query라도 사용자의 국가나 지역에 맞는 결과를 찾게 한다. 예를 들어 service availability, 현지 규정, 지원 전화번호와 매장 운영 시간은 지역 context가 없으면 관련성이 낮은 결과가 나올 수 있다.

이 기능은 결과 품질을 높이는 control이지 사실성을 보장하는 장치는 아니다. 허용된 domain에도 잘못되거나 오래된 content가 있을 수 있고 web page 자체가 prompt injection을 포함할 수 있으므로 source validation과 output evaluation이 여전히 필요하다.

## Domain Filtering 동작 방식

### Allowed Domains

`allowed_domains`를 사용하면 지정한 domain 안에서만 검색 결과를 가져온다.

적합한 사용 사례는 다음과 같다.

- AWS 기술 지원 agent가 AWS 공식 documentation과 blog만 검색
- 법률 assistant가 정부 및 승인된 법령 database만 검색
- 사내 research agent가 검증된 publisher와 partner site만 검색
- 제품 agent가 자사 support와 documentation domain만 검색

allow list는 unknown domain을 기본적으로 배제하므로 신뢰 경계가 분명하다. 대신 새 공식 domain이나 regional domain이 생길 때 list를 갱신하지 않으면 필요한 정보를 놓칠 수 있다.

### Blocked Domains

`blocked_domains`를 사용하면 일반 web 검색은 유지하면서 지정한 domain만 제외한다.

적합한 사용 사례는 다음과 같다.

- 검색 결과를 반복적으로 오염시키는 spam site 제외
- 사용 권한이나 compliance 문제로 이용할 수 없는 source 제외
- 품질 검증에서 낮은 점수를 받은 domain 임시 차단
- application 목적과 무관한 content network 제외

block list는 coverage가 넓지만 아직 평가하지 않은 새로운 domain이 기본적으로 허용된다. high-trust 업무에는 allow list가 더 보수적인 선택이다.

## User Location 동작 방식

application은 `web_search` tool에 선택적으로 location parameter를 제공한다. 검색 system은 이를 이용해 query의 geographic relevance를 높인다.

예를 들어 사용자가 "가까운 AWS 행사" 또는 "이 지역에서 사용할 수 있는 서비스"라고 물으면 location context가 없는 검색은 다른 국가의 결과를 반환할 수 있다. user location을 제공하면 해당 지역과 관련된 page를 더 우선적으로 찾을 수 있다.

location은 relevance hint로 이해해야 한다. application authorization, data residency enforcement 또는 법적 관할 판단을 대신하지 않는다. 정확한 규정 준수 여부는 공식 source와 별도의 policy logic으로 검증해야 한다.

## 서비스 및 아키텍처 관점

일반적인 request 흐름은 다음과 같다.

1. client가 질문과 필요한 경우 coarse-grained location을 application에 전달한다.
2. application backend가 user identity와 request policy를 확인한다.
3. use case별 search policy에서 allowed 또는 blocked domain 목록을 불러온다.
4. backend가 OpenAI Responses API 형식의 request에 Bedrock `web_search` tool과 filter를 구성한다.
5. Web Search가 domain 및 location 조건에 맞는 web result를 찾는다.
6. model이 검색 결과를 grounding context로 사용해 response를 생성한다.
7. application이 citation, source domain, freshness와 output policy를 검증한다.
8. user에게 answer와 source를 함께 제공한다.
9. query, 적용된 policy version, 사용한 source와 evaluation result를 monitoring system에 기록한다.

domain 목록을 prompt text에만 적지 말고 tool configuration의 structured parameter로 적용하는 것이 중요하다. prompt instruction은 model이 해석하지만 tool filter는 검색 단계에서 source 범위를 제한한다.

여러 업무를 하나의 agent가 처리한다면 use case별 policy를 분리한다. 예를 들어 기술 지원은 official documentation allow list를 사용하고 일반 시장 조사는 제한된 block list를 사용할 수 있다.

## 보안 관점

- 신뢰도가 중요한 workflow에는 deny list보다 allow list를 우선한다.
- 허용된 page도 indirect prompt injection을 포함할 수 있으므로 web content를 instruction이 아닌 untrusted data로 처리한다.
- 검색 결과가 agent tool execution, 결제 또는 infrastructure 변경을 직접 유발하지 않도록 사람의 승인과 policy check를 둔다.
- domain 이름의 typo, lookalike domain, subdomain과 redirect 처리 방식을 검증한다.
- user가 request에서 임의로 allow list를 확장하지 못하도록 server-side policy로 관리한다.
- location은 업무에 필요한 최소 범위만 수집하고 정확한 주소가 필요하지 않다면 국가나 도시 수준으로 줄인다.
- location과 query log에는 개인 정보가 포함될 수 있으므로 encryption, retention과 access control을 적용한다.
- response의 citation이 실제 주장을 뒷받침하는지 확인하고 source 없는 중요한 주장은 별도로 표시한다.
- 검색 source와 generated answer를 malware, unsafe link와 data exfiltration 관점에서 검사한다.
- tool invocation, filter와 selected source를 audit log에 남겨 incident 조사와 품질 평가에 활용한다.

## 운영 관점

domain policy는 application code에 흩어놓기보다 versioned configuration으로 중앙 관리하는 것이 좋다. owner, 변경 이유, 승인자와 적용 날짜를 기록하면 검색 결과 변화의 원인을 추적할 수 있다.

운영 metric에는 다음 항목을 포함할 수 있다.

- 검색 성공률과 no-result 비율
- 허용 또는 차단된 domain별 result 수
- answer citation coverage
- source freshness와 broken link 비율
- location 제공 전후 relevance score
- groundedness와 factual accuracy 평가
- search 및 model token 비용
- end-to-end latency
- user correction과 escalation 비율

allow list가 너무 좁으면 no-result와 오래된 답변이 늘 수 있고 block list가 너무 느슨하면 source 품질 편차가 커질 수 있다. offline evaluation set과 production feedback을 이용해 policy를 조정해야 한다.

cache를 사용할 때는 query뿐 아니라 domain filter, location과 policy version을 cache key에 포함해야 한다. 그렇지 않으면 다른 지역이나 다른 trust policy의 결과가 잘못 재사용될 수 있다.

이 기능은 Web Search가 지원되는 US East (N. Virginia), US East (Ohio), US West (Oregon), AWS GovCloud (US-West) Region에서 제공된다. 사용 전 대상 Region과 pricing을 확인해야 한다.

## 도입 전 확인할 질문

1. 업무상 반드시 신뢰해야 하는 source는 무엇인가?
2. allow list와 block list 중 어느 방식이 위험 수준에 맞는가?
3. regional 또는 language별 subdomain도 목록에 포함해야 하는가?
4. redirect와 syndicated content를 어떻게 처리할 것인가?
5. location은 국가, Region, 도시 중 어느 정도 정밀도가 필요한가?
6. 사용자의 location 제공 동의와 retention policy가 준비되어 있는가?
7. web content의 prompt injection을 어떻게 탐지하고 격리할 것인가?
8. citation correctness와 source freshness를 어떤 evaluation으로 측정할 것인가?
9. 검색 실패 시 답변 거절, 기존 knowledge base 또는 사람 escalation 중 어떤 fallback을 사용할 것인가?

## 시험 관점 정리

- Amazon Bedrock은 foundation model과 생성형 AI application 구축 기능을 제공하는 managed service다.
- grounding은 model response를 외부 source의 정보에 연결하는 과정이다.
- allow list는 명시적으로 허용한 대상만 사용하게 하는 default-deny 방식이다.
- block list는 명시적으로 거부한 대상만 제외하는 default-allow 방식이다.
- OpenAI Responses API는 model response와 tool use를 구성하는 API 형식이다.
- web search tool은 최신 web content를 retrieval해 model context에 제공한다.
- user location은 검색 relevance를 높이는 context이며 authorization control이 아니다.
- prompt injection은 외부 content의 instruction이 model behavior를 바꾸려는 공격이다.
- structured tool parameter는 자연어 prompt보다 일관된 control을 제공한다.
- citation은 answer의 근거 source를 추적하게 하지만 source의 정확성을 자동 보장하지 않는다.
- data minimization은 목적에 필요한 최소 개인 정보만 수집하는 원칙이다.
- cache key에는 결과에 영향을 주는 filter와 location context도 포함해야 한다.

## 같이 보면 좋은 학습 포인트

- Amazon Bedrock tool use
- OpenAI Responses API 구조
- retrieval-augmented generation과 web grounding
- allow list와 deny list security model
- indirect prompt injection 방어
- citation correctness 평가
- groundedness와 relevance metric
- user location privacy와 data minimization
- Region별 Bedrock feature availability
- search result caching과 policy versioning
- human-in-the-loop approval
- observability for generative AI

## 한 줄 정리

Bedrock Web Search의 domain filtering과 user location control은 agent가 신뢰할 source와 지역 관련 결과를 더 정확히 선택하게 하지만, allow list 관리, prompt injection 방어, 위치 정보 최소화와 citation 평가를 함께 적용해야 안전한 grounding이 된다.
