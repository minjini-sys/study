# Amazon SageMaker HyperPod Inference Gateway 출시

- 정리 날짜: 2026-09-21
- 원문 게시일: 2026-09-18
- 공식 출처: [AWS Machine Learning Blog - Introducing Amazon SageMaker HyperPod Inference Gateway](https://aws.amazon.com/blogs/machine-learning/introducing-amazon-sagemaker-hyperpod-inference-gateway/)

## 핵심 내용

AWS가 Amazon SageMaker HyperPod 환경에서 대규모 언어 모델 추론 요청을 GPU 상태에 따라 지능적으로 분배하는 Inference Gateway를 발표했다. 이 기능은 기존 HyperPod 기반 Amazon EKS 클러스터에 관리형 애드온으로 설치되며, 모델 서버나 클라이언트 애플리케이션 코드를 변경하지 않고 사용할 수 있다.

기존 Kubernetes의 라운드 로빈 방식은 GPU의 KV 캐시 사용률, 요청 대기열, LoRA 어댑터 적재 여부를 알지 못한다. Inference Gateway는 실시간 GPU 지표를 활용해 각 요청에 가장 적합한 Pod를 선택한다. AWS가 공개한 테스트에서는 워크로드에 따라 첫 토큰 지연 시간(TTFT)이 크게 줄었으며, 대표적으로 사용자가 4.4초 기다리던 경우 800ms 미만으로 단축됐다.

## 왜 중요한가

생성형 AI 추론에서는 GPU가 가장 비싼 자원 중 하나다. 일부 Pod에 요청이 몰리는 동안 다른 GPU가 쉬고 있으면 지연 시간과 비용이 동시에 증가한다. 특히 서로 다른 GPU 세대가 섞인 클러스터, 순간 트래픽이 큰 서비스, 공통 프롬프트 접두사가 반복되는 서비스에서는 단순한 연결 수 기반 라우팅만으로 자원을 효율적으로 쓰기 어렵다.

Inference Gateway는 모델과 GPU 내부 상태를 라우팅 결정에 반영한다. 따라서 과도한 GPU 증설 없이도 응답 속도와 처리량을 개선할 가능성이 있으며, 운영팀이 직접 복잡한 추론 스케줄러를 개발하고 유지하는 부담도 줄어든다.

## 서비스 및 아키텍처 관점

현재 제공되는 Tier 1은 각 HyperPod/EKS 클러스터에 설치되는 구조이며 세 가지 핵심 구성 요소로 이루어진다.

1. Envoy Gateway가 HTTPS 요청을 받고 클러스터의 단일 비공개 엔드포인트를 제공한다.
2. Body-Based Router가 OpenAI 호환 요청의 `model` 값을 읽어 올바른 모델 풀로 전달한다.
3. Endpoint Picker가 Prometheus 지표를 바탕으로 가장 적합한 모델 Pod를 선택한다.

Endpoint Picker는 KV 캐시 사용률, 대기열 깊이, 실행 중인 요청 수, LoRA 어댑터 적재 여부, 프롬프트 접두사 캐시 적중 가능성을 종합한다. 각 지표의 가중치를 조절해 지연 시간 중심의 대화형 서비스나 처리량 중심의 배치 작업에 맞출 수 있다.

AWS는 향후 여러 클러스터와 리전을 조정하는 Tier 2 Global Inference Router도 예고했다. 이 계층에는 교차 클러스터 장애 조치, 전역 속도 제한, 비용을 고려한 트래픽 분배가 포함될 예정이다.

## 보안 및 운영 관점

- 게이트웨이는 클러스터마다 단일 비공개 엔드포인트를 제공하므로 VPC와 보안 그룹 정책을 함께 설계한다.
- EKS 애드온과 CRD는 GitOps로 버전 관리하고 변경 검토 및 롤백 절차를 마련한다.
- 모델별 접근 권한과 테넌트 분리 정책을 라우팅 설정과 별도로 적용한다.
- Pod 수준의 KV 캐시, 대기열, 어댑터 지표와 클러스터 수준의 오류율 및 P99 지연 시간을 함께 관찰한다.
- 풀이 고갈되어 HTTP 429가 발생할 때 재시도 정책과 자동 확장이 함께 동작하도록 구성한다.
- 특정 모델이나 LoRA 어댑터에 요청이 집중될 때 GPU 메모리와 비용 변화를 점검한다.

## 시험 관점 정리

- Amazon SageMaker HyperPod는 대규모 AI 학습 및 추론 인프라를 운영하기 위한 서비스다.
- Amazon EKS 관리형 애드온은 설치, 업그레이드, 롤백의 운영 부담을 낮춘다.
- CloudWatch와 Prometheus는 서로 다른 계층의 성능 및 운영 지표를 관찰하는 데 활용된다.
- 자동 확장만 적용하기 전에 병목의 원인이 용량 부족인지 비효율적인 라우팅인지 구분해야 한다.
- 다중 AZ와 다중 리전 설계에서는 성능뿐 아니라 장애 조치 시간과 비용을 함께 고려한다.

## 같이 보면 좋은 학습 포인트

- Kubernetes Gateway API와 Inference Extension
- Envoy의 L7 프록시 및 라우팅 역할
- LLM의 KV 캐시와 첫 토큰 지연 시간(TTFT)
- LoRA 어댑터를 공유 GPU 환경에서 운영하는 방법
- Prometheus, Grafana, Amazon CloudWatch를 이용한 추론 관측성
- SageMaker HyperPod와 일반 SageMaker 관리형 엔드포인트의 선택 기준

## 한 줄 정리

SageMaker HyperPod Inference Gateway는 실시간 GPU 상태와 모델 캐시 정보를 바탕으로 추론 요청을 적절한 Pod에 보내 GPU 낭비와 응답 지연을 줄이는 Kubernetes 기반 라우팅 계층이다.
