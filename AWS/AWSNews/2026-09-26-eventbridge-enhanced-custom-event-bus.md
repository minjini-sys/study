# Amazon EventBridge 향상된 Custom Event Bus 출시

- 정리 날짜: 2026-09-26
- 원문 게시일: 2026-09-24
- 공식 출처: [AWS News Blog - Introducing enhanced custom event buses in Amazon EventBridge for enterprise-scale event-driven applications](https://aws.amazon.com/blogs/aws/introducing-enhanced-custom-event-buses-in-amazon-eventbridge-for-enterprise-scale-event-driven-applications/)

## 핵심 내용

AWS가 여러 팀과 AWS 계정에서 사용하는 대규모 이벤트 기반 애플리케이션을 위한 향상된 Amazon EventBridge Custom Event Bus를 출시했다. 하나의 중앙 이벤트 버스를 AWS Organizations의 여러 계정과 공유하고, 각 팀이 같은 버스에서 독립적으로 이벤트를 발행하거나 구독할 수 있다.

새 버스는 순서 보장 전송, 단순화된 Subscriber 리소스, 이벤트 재생, 콘텐츠 기반 중복 제거, Avro 및 Protocol Buffers 역직렬화와 새로운 처리량 기반 요금 체계를 제공한다. 기존 Custom Event Bus는 `Custom event bus - classic`으로 계속 동작하며 새 버스로 즉시 이전할 필요는 없다.

## 왜 중요한가

조직이 멀티 계정 구조로 성장하면 팀마다 이벤트 버스를 만들고 교차 계정 규칙이나 버스 간 라우팅을 구성하게 된다. 이 방식은 이벤트 흐름을 파악하기 어렵고, 라우팅 비용이 중첩되며, 새로운 구독자를 추가할 때 플랫폼 팀의 작업을 기다려야 하는 문제가 있다.

향상된 버스는 조직 전체의 중앙 이벤트 백본을 만들면서도 발행자와 구독자의 결합도를 낮춘다. 플랫폼 팀은 전체 흐름과 권한을 통제하고, 애플리케이션 팀은 필요한 이벤트 구독을 직접 정의할 수 있다.

## 서비스 및 아키텍처 관점

향상된 Custom Event Bus의 기본 흐름은 다음과 같다.

1. 플랫폼 계정에 중앙 EventBridge Custom Event Bus를 생성한다.
2. AWS Resource Access Manager(AWS RAM)로 조직, OU, 계정 또는 IAM 주체와 버스를 공유한다.
3. 각 애플리케이션이 같은 버스로 이벤트를 발행한다.
4. 소비 팀이 Subscriber 리소스에 필터, 대상, 재시도 정책, DLQ를 함께 정의한다.
5. EventBridge가 이벤트 보존, 필터링, 변환, 라우팅 및 전송을 관리한다.

한 버스에는 기본적으로 최대 10,000개의 Subscriber를 만들 수 있으며 할당량 증가를 요청할 수 있다. Subscriber는 JSONata로 대상에 전달할 이벤트 구조를 변환할 수 있고, 시작 시점을 선택해 새 애플리케이션의 데이터 초기화나 장애 복구용 이벤트 재생에 활용할 수 있다.

## 순서 보장과 중복 제거

발행자가 이벤트에 `EventGroupId`를 포함하면 같은 그룹의 이벤트를 순서대로 전달할 수 있다. 예를 들어 배송 차량별 위치 업데이트에 차량 ID를 그룹으로 사용하면 각 차량의 이벤트 순서를 유지하면서 다른 소비자는 같은 이벤트를 비순서 방식으로 처리할 수 있다.

콘텐츠 기반 중복 제거를 활성화하면 EventBridge가 이벤트의 의미 있는 부분을 해시해 5분 이내에 들어온 같은 이벤트를 제거한다. 자체 멱등성 키를 사용할 수 없는 발행자의 재시도 처리에 유용하다.

## 보안 및 운영 관점

- AWS Organizations와 AWS RAM 공유 범위를 필요한 조직 단위와 계정으로 제한한다.
- 이벤트 발행 및 구독 권한을 IAM 최소 권한 정책으로 분리한다.
- 민감 정보는 이벤트 본문에 직접 넣지 않고 참조 식별자와 안전한 데이터 저장소를 사용한다.
- Subscriber별 재시도 정책과 Dead-Letter Queue를 구성하고 실패 이벤트를 경보로 연결한다.
- 이벤트 스키마와 버전을 관리해 발행자 변경이 소비자를 깨뜨리지 않도록 한다.
- 순서 보장이 필요한 이벤트와 높은 처리량이 필요한 이벤트를 구분해 설계한다.
- 발행자와 구독자에 태그와 비용 할당 기준을 적용해 처리량 기반 비용을 추적한다.

## 시험 관점 정리

- Amazon EventBridge는 이벤트 소스와 대상을 느슨하게 연결하는 서버리스 이벤트 버스 서비스다.
- AWS RAM은 AWS 리소스를 조직, OU 또는 다른 계정과 공유하는 데 사용한다.
- 비동기 시스템은 중복 전송과 실패를 고려해 멱등성과 DLQ를 설계해야 한다.
- 순서 보장은 필요한 이벤트 그룹에만 적용해 확장성과 처리량을 유지한다.
- EventBridge 규칙은 이벤트 패턴으로 필요한 이벤트만 대상으로 라우팅한다.
- 이벤트 재생은 장애 복구, 새 소비자의 상태 초기화, 재처리에 활용할 수 있다.

## 같이 보면 좋은 학습 포인트

- 이벤트 기반 아키텍처와 발행-구독 패턴
- EventBridge Event Bus, Rule, Target의 관계
- AWS Organizations와 AWS RAM
- Amazon SQS FIFO와 EventBridge 순서 보장의 선택 기준
- 멱등성, 재시도, DLQ 설계
- Schema Registry와 이벤트 버전 관리
- JSONata를 이용한 이벤트 변환

## 한 줄 정리

향상된 Amazon EventBridge Custom Event Bus는 조직 전체가 공유하는 중앙 이벤트 백본에 순서 보장, 독립적인 Subscriber, 중복 제거와 재생 기능을 더해 멀티 계정 이벤트 아키텍처를 단순화한다.
