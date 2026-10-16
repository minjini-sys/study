# Amazon Bedrock AgentCore로 구축하는 운영용 AI 에이전트 평가 프레임워크

- 정리 날짜: 2026-10-17
- 원문 게시일: 2026-10-16
- 공식 출처: [AWS Artificial Intelligence Blog - Building trust in AI agents: A production-ready evaluation framework with Amazon Bedrock AgentCore](https://aws.amazon.com/blogs/machine-learning/building-trust-in-ai-agents-a-production-ready-evaluation-framework-with-amazon-bedrock-agentcore/)

## 핵심 내용

AWS가 Amazon Bedrock AgentCore Evaluations를 중심으로 AI agent의 개발부터 production 운영과 개선까지 이어지는 9단계 평가 프레임워크를 공개했다.

일반적인 monitoring은 latency, error rate와 availability를 확인한다. 하지만 AI agent는 빠르게 정상 응답하면서도 틀린 숫자, 근거 없는 계산이나 다른 고객의 정보를 자신 있게 제시할 수 있다. 이런 silent quality degradation은 infrastructure metric만으로 탐지하기 어렵다.

이 프레임워크는 safety, quality, performance, cost와 operational control의 다섯 차원을 지속해서 평가한다. tool, 개별 sub-agent와 multi-agent orchestration의 세 계층마다 별도의 golden set, evaluator와 promotion 기준을 적용한다.

## 왜 중요한가

AI agent는 동일한 입력에도 다른 reasoning path, tool과 결과를 선택할 수 있다. 하나의 tool selection 오류가 이후 retrieval, 계산과 synthesis를 연쇄적으로 오염시킬 수도 있다.

AWS 내부 sales intelligence agent의 사전 평가에서는 410.08달러를 410.08 million dollar로 표시하거나, 존재하지 않는 annual recurring revenue를 만들어 내고 자체 계산을 source fact처럼 제시하는 문제가 발견됐다. 당시 일반 monitoring 상태는 정상으로 보였다.

또한 data, tool과 prompt가 함께 release되어야 하는 상황에서 일부만 먼저 반영되자 평가 pass rate가 78%에서 38%로 떨어졌다. 이 사례는 agent 품질 검증이 최종 QA가 아니라 release gate와 production monitoring에 포함되어야 함을 보여준다.

## 다섯 가지 평가 차원

- Safety: agent가 policy, data access와 행동 범위를 지키는가
- Quality: 답변이 정확하고 관련성이 높으며 source에 근거하는가
- Performance: tool 및 전체 workflow의 latency와 reliability가 목표를 만족하는가
- Cost: query당 token, model, tool과 infrastructure 비용이 budget 안에 있는가
- Operational control: 문제를 빠르게 탐지하고 원인과 영향을 추적해 복구할 수 있는가

각 차원은 독립적으로 보지 않는다. 정확하지만 너무 늦게 도착하는 답변, 안전하지만 근거가 없는 답변, 품질은 좋지만 budget을 지속해서 초과하는 workflow도 production 기준을 만족하지 못한다.

## 9단계 평가 프레임워크

### 1. Foundation: 좋은 결과 정의

agent가 제공해야 할 business outcome을 먼저 정의하고 이를 측정 가능한 metric으로 변환한다. failure taxonomy, evaluator 구성, golden set, safety rule, latency budget과 cost target이 이후 모든 단계의 contract가 된다.

### 2. Experiment: 더 나은 version 선택

prompt, model, retrieval strategy와 tool description 후보를 같은 dataset과 evaluator로 비교한다. 평균 점수뿐 아니라 failure category, latency와 query cost도 함께 평가한다.

### 3. Merge: CI regression 확인

code, prompt 또는 tool 변경을 merge하기 전에 golden set을 실행한다. 기존 version보다 accuracy, faithfulness나 tool selection이 나빠지면 build를 실패시킨다.

### 4. Gate: production 배포 판단

candidate revision이 quality, safety, latency와 cost threshold를 모두 통과해야 배포할 수 있다. data pipeline, prompt와 application code처럼 함께 배포해야 하는 component의 compatibility도 검사한다.

### 5. Canary: 조용한 장애 탐지

일부 traffic을 candidate agent에 보내 현재 production version과 점수를 비교한다. 알려진 test case뿐 아니라 실제 사용자의 long-tail 표현에서도 품질이 유지되는지 확인한다.

### 6. Adversary: 공격 상황 평가

prompt injection, policy bypass, unauthorized data access와 악의적인 tool input을 시험한다. safety evaluation은 이 단계에만 존재하는 것이 아니라 모든 단계와 요청에 적용된다.

### 7. Drift: production 품질 변화 탐지

model release가 없어도 source data와 사용자의 질문 분포가 바뀌면 품질이 저하될 수 있다. rolling window로 score를 계산하고 threshold를 연속으로 벗어나면 alert를 발생시킨다.

### 8. Pulse: 실시간 품질 가시성

AgentCore Observability가 reasoning step, tool call 순서, input/output, latency와 token 사용량을 OpenTelemetry trace로 수집한다. CloudWatch alarm에서 문제가 발생한 agent trace로 바로 이동할 수 있어 root cause 분석 시간을 줄인다.

### 9. Feedback Loop: 지속적 개선

낮은 점수의 production session을 golden set에 추가하고 반복되는 failure pattern을 prompt, tool description와 routing 개선에 사용한다. AgentCore optimization은 trace와 target evaluator를 분석해 system prompt 및 tool description recommendation을 생성하고 A/B test로 결과를 검증한다.

## Evaluator 구성

하나의 evaluator로 모든 실패를 잡을 수 없으므로 서로 다른 방식을 결합한다.

- AgentCore built-in evaluator: relevance, faithfulness와 helpfulness 같은 일반 품질 평가
- Ground-truth evaluator: 알려진 정답, entity, metric과 기간을 deterministic하게 비교
- LLM-as-judge: 하나의 정확한 문자열 답이 없는 reasoning 및 응답 품질을 rubric으로 평가
- Custom evaluator: domain-specific rule과 failure taxonomy를 AWS Lambda 등으로 구현

정확한 숫자와 entity는 deterministic evaluator가 유리하고, 설명의 유용성이나 reasoning 품질은 LLM-as-judge가 더 넓게 평가할 수 있다. 중요한 결과에는 human review sample을 유지해 evaluator 자체의 bias와 drift도 점검해야 한다.

## 서비스 및 아키텍처 관점

1. application과 multi-agent orchestration이 AgentCore Runtime에서 실행된다.
2. AgentCore Gateway가 tool과 candidate/current agent 사이의 traffic을 관리한다.
3. AgentCore Evaluations가 on-demand, batch와 online evaluation을 수행한다.
4. Amazon Bedrock Guardrails가 input의 prompt injection과 output의 policy 및 sensitive data 위반을 검사한다.
5. AgentCore Observability와 OpenTelemetry가 reasoning 및 tool span을 CloudWatch에 전달한다.
6. CloudWatch metric, log와 alarm이 품질, latency, token 및 cost threshold를 감시한다.
7. evaluation output과 전체 trace는 Amazon S3 및 Redshift에 장기 보관한다.
8. Amazon Quick dashboard가 agent, release와 기간별 score trend를 시각화한다.
9. Lambda custom evaluator가 business-specific accuracy와 policy rule을 계산한다.
10. AgentCore optimization이 낮은 점수의 trace로 prompt와 tool description 개선안을 만든다.

evaluation score pipeline과 execution observability pipeline은 서로 다르다. 실행 trace가 정상이라고 답변이 정확한 것은 아니므로 두 데이터를 agent session 또는 trace identifier로 연결해 분석해야 한다.

## 보안 및 운영 관점

- input과 output 모두에 Guardrails를 적용하고 production traffic 전체를 검사한다.
- role 및 territory별 data access control을 agent 밖의 authoritative system에서도 강제한다.
- evaluation dataset에는 실제 sensitive data 대신 masking 또는 synthetic case를 사용한다.
- evaluator와 LLM-as-judge가 다른 고객 정보나 confidential source를 prompt에 포함하지 않도록 한다.
- tool별 IAM role을 분리하고 agent에는 필요한 action만 허용한다.
- tool input/output과 reasoning trace의 보관 기간 및 접근 권한을 정한다.
- prompt, model, tool, data schema와 evaluator version을 함께 기록해 결과를 재현한다.
- pass/fail threshold는 평균뿐 아니라 p90/p99 latency와 중요 failure type을 포함한다.
- canary rollout에는 자동 중단 및 이전 revision 복귀 조건을 설정한다.
- production feedback을 golden set으로 승격할 때 PII와 user consent를 검토한다.
- evaluation 자체의 model 및 token 비용을 budget과 sampling policy로 관리한다.

## 운영 성과 사례

실제 traffic에서 customer identification accuracy가 연속된 30분 구간 동안 98%에서 81%, 다시 80%로 하락했다. CloudWatch alarm과 AgentCore trace를 조사한 결과 stock ticker, shorthand name과 typo가 기존 synthetic golden set에 포함되지 않은 것이 원인이었다.

낮은 점수의 session을 golden set에 추가하고 disambiguation tool이 입력 형식을 normalize하도록 수정한 뒤 다음 evaluation cycle에서 baseline을 회복했다. 이 사례는 production drift를 regression test로 다시 흡수하는 closed loop를 보여준다.

AWS 사례에서는 실시간 품질 가시성 도입 후 mean time to detect가 수일 또는 수주에서 4시간 미만으로 줄었다. prompt의 few-shot example에 실제 숫자와 고객명이 포함되어 발생한 hallucination을 placeholder로 바꾸자 1만 응답당 순수 hallucination이 138건에서 52건 미만으로 줄고 grounding rate는 93%에서 96%로 높아졌다.

## 시험 관점 정리

- Amazon Bedrock AgentCore는 agent runtime, gateway, memory, identity, observability와 evaluation 같은 production 기능을 제공한다.
- Amazon Bedrock Guardrails는 harmful content, denied topic, sensitive information과 prompt attack 방어를 적용한다.
- golden set은 반복 평가에 사용할 대표 입력과 기대 결과 모음이다.
- regression test는 기존에 통과한 동작이 변경 후 깨지지 않았는지 확인한다.
- canary deployment는 새 version에 일부 production traffic만 보내 위험을 제한한다.
- drift는 input, data 또는 output 분포가 시간에 따라 변하는 현상이다.
- faithfulness는 답변의 주장이 제공된 source context에 근거하는지를 평가한다.
- LLM-as-judge는 language model이 rubric에 따라 다른 model 또는 agent의 결과를 채점하는 방식이다.
- OpenTelemetry는 trace, metric과 log를 위한 vendor-neutral observability standard다.
- A/B test는 두 version을 실제 또는 대표 traffic에서 비교해 더 나은 결과를 선택한다.

## 같이 보면 좋은 학습 포인트

- Amazon Bedrock AgentCore Evaluations
- AgentCore Observability와 OpenTelemetry
- Amazon Bedrock Guardrails
- deterministic evaluator와 LLM-as-judge 비교
- golden dataset 설계와 failure taxonomy
- CI/CD quality gate 및 canary deployment
- AI agent drift detection
- multi-agent routing과 tool selection accuracy
- CloudWatch metric, alarm과 trace correlation
- AgentOps, MLOps와 DevOps의 차이

## 한 줄 정리

AgentCore 기반 9단계 프레임워크는 tool, sub-agent와 multi-agent 전체에서 안전·정확성·성능·비용·운영 통제를 개발부터 production까지 지속적으로 평가하고 실제 실패를 다음 regression test와 개선으로 연결한다.
