# Amazon CloudWatch Omni 출시: AI 에이전트를 위한 통합 관측성

- 정리 날짜: 2026-09-23
- 원문 게시일: 2026-09-22
- 공식 출처: [AWS News Blog - Introducing Amazon CloudWatch Omni: AI-powered observability for generative AI and agentic workloads](https://aws.amazon.com/blogs/aws/introducing-amazon-cloudwatch-omni-ai-powered-observability-for-generative-ai-and-agentic-workloads/)

## 핵심 내용

AWS가 애플리케이션과 생성형 AI 에이전트의 관측성, 평가, 실험을 하나로 묶은 Amazon CloudWatch Omni를 정식 출시했다. Omni는 특정 모델 공급자, 에이전트 프레임워크, 실행 환경에 종속되지 않으며 OpenTelemetry 계열의 개방형 표준을 기반으로 한다.

개발자는 VS Code와 Kiro 확장에서 에이전트 실행 추적을 확인하고, 운영자는 AWS Management Console과 분리된 웹 화면에서 전체 에이전트와 애플리케이션을 관찰할 수 있다. 두 화면은 같은 추적 데이터를 공유하므로 개발 중 발견한 문제와 운영 환경의 장애를 같은 맥락에서 분석할 수 있다.

Omni는 LLM 호출, 도구 실행, 추론 단계, 토큰 사용량, 지연 시간을 구조화된 타임라인으로 기록한다. 17개의 기본 평가기를 통해 일관성, 유용성, 충실도, 라우팅 정확성 등을 측정하고 프롬프트나 모델 구성을 나란히 비교할 수 있다.

## 왜 중요한가

일반 애플리케이션은 오류율과 지연 시간이 정상이라면 서비스가 정상이라고 판단할 수 있는 경우가 많다. 하지만 AI 에이전트는 같은 입력에도 다른 경로를 선택할 수 있고, 프롬프트 변경만으로 답변 품질이나 도구 선택이 나빠질 수 있다. 기존 인프라 지표만으로는 이러한 품질 저하를 발견하기 어렵다.

CloudWatch Omni는 운영 지표와 에이전트 품질 평가를 연결한다. 팀은 실제 운영 추적으로 테스트 데이터셋을 만들고, 변경 전후의 품질과 비용, 지연 시간을 비교해 회귀 문제를 배포 전에 확인할 수 있다.

## 서비스 및 아키텍처 관점

CloudWatch Omni의 흐름은 다음과 같다.

1. 애플리케이션과 에이전트에 OpenInference 또는 ADOT 기반 계측을 추가한다.
2. LLM 호출과 도구 실행이 분산 추적의 span으로 기록된다.
3. 로컬 개발 단계에서는 데이터를 로컬에 저장해 IDE에서 확인할 수 있다.
4. 선택적으로 Cloud Login을 연결하면 추적을 Amazon CloudWatch에 지속 저장하고 팀과 공유한다.
5. Trace Explorer, Session Explorer, Agent Topology로 실행 흐름과 병목을 분석한다.
6. 평가기와 실험 기능으로 프롬프트, 모델, 설정 조합의 품질과 성능을 비교한다.

Omni는 LangChain, LangGraph, CrewAI, OpenAI SDK, Strands, Vercel AI SDK 등을 지원하며 Python과 TypeScript에서 사용할 수 있다. 에이전트는 AWS Lambda, Amazon ECS, Amazon EKS 또는 다른 클라우드에서 실행할 수 있다.

## 보안 및 운영 관점

- 프롬프트, 응답, 도구 입출력에 개인정보나 비밀 값이 기록될 수 있으므로 수집 범위와 마스킹 정책을 먼저 정한다.
- CloudWatch에 전송되는 추적 데이터에는 최소 권한 IAM 정책과 적절한 보존 기간을 적용한다.
- 웹 환경은 기업 SSO를 사용하고 개발자에게 AWS 콘솔 권한을 불필요하게 부여하지 않는다.
- 토큰 사용량, 지연 시간, 오류율과 함께 평가 점수의 하락을 경보 기준으로 활용한다.
- 운영 트래픽에서 만든 평가 데이터셋은 민감 정보를 제거하고 접근 권한을 제한한다.
- 프롬프트 버전을 관리하고 품질이 저하된 버전으로부터 빠르게 롤백할 수 있게 한다.

## 시험 관점 정리

- Amazon CloudWatch는 지표, 로그, 추적을 수집하고 분석하는 AWS 관측성 서비스다.
- 분산 추적은 여러 서비스와 도구 호출로 이어지는 요청의 전체 경로를 이해하는 데 사용한다.
- OpenTelemetry는 벤더에 종속되지 않는 관측 데이터 수집 표준이다.
- 생성형 AI 운영에서는 가용성과 지연 시간뿐 아니라 정확성, 충실도, 토큰 비용 같은 품질 지표도 필요하다.
- 최소 권한, 데이터 보존, 암호화, 민감 정보 마스킹은 관측성 파이프라인에도 동일하게 적용된다.

## 같이 보면 좋은 학습 포인트

- OpenTelemetry의 trace, span, metric, log 개념
- AWS Distro for OpenTelemetry(ADOT)
- Amazon CloudWatch Logs와 X-Ray의 분산 추적
- AI 에이전트 평가용 golden dataset과 회귀 테스트
- 프롬프트 버전 관리 및 A/B 실험
- Amazon Bedrock AgentCore Observability와 CloudWatch Omni의 연계

## 한 줄 정리

Amazon CloudWatch Omni는 AI 에이전트의 실행 경로와 답변 품질을 함께 추적하고 비교해 프롬프트 및 모델 변경으로 생기는 회귀를 발견하도록 돕는 통합 관측성 서비스다.
