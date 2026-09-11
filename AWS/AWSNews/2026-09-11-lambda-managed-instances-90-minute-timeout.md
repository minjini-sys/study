# AWS Lambda Managed Instances에서 90분 함수 타임아웃 지원

날짜: 2026-09-11  
출처: [AWS Compute Blog - Announcing 90-minute function timeout on AWS Lambda Managed Instances](https://aws.amazon.com/blogs/compute/)

## 오늘의 AWS 소식

AWS Compute Blog에 따르면 AWS Lambda Managed Instances가 비동기 호출과 이벤트 소스 매핑(Event Source Mapping, ESM) 호출에서 최대 90분 함수 타임아웃을 지원한다.

기존 Lambda의 일반적인 최대 실행 시간은 15분으로 이해하는 경우가 많았는데, Lambda Managed Instances에서는 특정 호출 유형에 대해 이 제한이 6배 늘어난 것이다.

## 핵심 내용

이번 업데이트의 핵심은 장시간 실행되는 작업을 Lambda 기반으로 처리할 수 있는 선택지가 늘었다는 점이다.

AWS가 언급한 대표적인 사용 사례는 다음과 같다.

- 데이터 처리
- 미디어 트랜스코딩
- AI 추론
- 배치 워크로드

기존에는 이런 작업이 15분 제한에 걸리면 Amazon ECS, AWS Batch, Amazon EC2, Step Functions 분할 처리 등을 고려해야 했다.

이번 업데이트로 일부 장시간 작업은 Lambda Managed Instances만으로도 처리 가능성이 생겼다.

## 왜 중요한가

서버리스 아키텍처에서는 실행 시간 제한이 설계에 직접적인 영향을 준다.

15분 안에 끝나지 않는 작업은 보통 다음 방식으로 우회했다.

- 작업을 여러 작은 단위로 쪼갠다.
- Step Functions로 상태를 나눠 관리한다.
- SQS와 Lambda를 조합해 재시도 구조를 만든다.
- ECS, Batch, EC2 같은 장시간 실행 컴퓨팅으로 넘긴다.

90분 타임아웃은 이런 설계 복잡도를 줄일 수 있다.

다만 모든 상황에서 긴 타임아웃이 좋은 선택은 아니다.

실행 시간이 길어질수록 장애 복구, 재시도, 비용, 중복 실행, 관찰 가능성 설계가 더 중요해진다.

## 아키텍처 관점에서 주의할 점

장시간 Lambda 실행을 설계할 때는 다음 항목을 확인해야 한다.

- 함수가 멱등성을 갖는지
- 중간 실패 시 재시도 전략이 있는지
- 실행 로그와 메트릭을 충분히 남기는지
- 타임아웃 직전 상태를 복구할 수 있는지
- 전체 비용이 ECS, Batch, EC2보다 합리적인지
- 외부 API나 데이터베이스 연결이 긴 실행 시간을 견딜 수 있는지

특히 비동기 호출은 재시도와 중복 실행 가능성을 고려해야 한다.

이벤트 소스 매핑을 사용하는 경우에는 소스 서비스의 재처리 방식, 배치 크기, 실패 레코드 처리 전략도 같이 봐야 한다.

## 시험 관점에서 정리

AWS 자격증 관점에서는 단순히 "Lambda가 90분 가능"이라고 외우는 것보다 서비스 선택 기준을 이해하는 것이 중요하다.

정리하면 다음과 같다.

- 짧고 이벤트 기반 작업에는 Lambda가 적합하다.
- 장시간 작업에는 ECS, Batch, EC2, Step Functions도 후보가 된다.
- Lambda Managed Instances의 90분 지원은 일부 장시간 비동기/ESM 워크로드에서 서버리스 선택지를 넓힌다.
- 긴 실행 시간은 비용과 장애 복구 설계를 반드시 같이 검토해야 한다.

## 같이 보면 좋은 학습 포인트

AWS Skill Builder에서는 다음 주제를 함께 보면 좋다.

- AWS Lambda
- Amazon EventBridge
- Amazon SQS
- AWS Step Functions
- AWS Batch
- 서버리스 아키텍처 설계

참고: [AWS Skill Builder](https://skillbuilder.aws/)

## 한 줄 정리

<mark>AWS Lambda Managed Instances가 비동기 및 이벤트 소스 매핑 호출에서 최대 90분 타임아웃을 지원하면서, 장시간 데이터 처리·AI 추론·배치 작업을 서버리스로 처리할 수 있는 선택지가 넓어졌다.</mark>
