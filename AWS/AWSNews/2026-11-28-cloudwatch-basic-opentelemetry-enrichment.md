# Amazon CloudWatch, 기본 OpenTelemetry 보강 지원

- 정리일 2026-11-28
- 발표일 2026-11-27
- [AWS What's New 공식 발표](https://aws.amazon.com/about-aws/whats-new/2026/11/amazon-cloudwatch-opentelemetry-enrichment/)

## 핵심 내용

Amazon CloudWatch가 AWS 서비스 지표에 기본 OpenTelemetry 보강을 추가했다. EC2의 `CPUUtilization`에 붙는 `InstanceId`처럼 리소스 식별 차원이 있는 지원 지표를 OpenTelemetry 데이터 포인트로 제공해 PromQL로 조회할 수 있게 한다. 기본 보강에는 추가 요금이 없다.

보강된 지표에는 리소스 ARN, 클라우드 공급자, 리전, 계정 ID가 리소스 속성으로 추가된다. 원래 CloudWatch 차원은 데이터 포인트 속성으로 표현되며, 계측 범위(instrumentation scope)에 지표를 게시한 AWS 서비스가 표시된다. 기존 CloudWatch API에서도 원래 지표를 계속 조회할 수 있다. PromQL에서 AWS 리소스 태그를 라벨로 조회하려면 별도의 유료 리소스 태그 보강을 선택한다.

## 왜 중요한가

PromQL로 애플리케이션 지표를 보는 운영팀은 같은 쿼리 언어로 AWS 리소스 지표도 조회할 수 있다. 예를 들어 애플리케이션의 응답 지연이 늘어난 시간대에 해당 리전의 EC2 CPU 사용률을 함께 살펴보면 조사 범위를 좁힐 수 있다.

## 서비스·아키텍처 관점

AWS 서비스가 CloudWatch에 지표를 게시하면 기본 보강이 지원 지표에 리소스 정보를 붙이고, 대시보드는 PromQL로 조회한다. 계정·리전·인스턴스 식별자로 집계하거나 필터링하는 대시보드부터 적용해 볼 수 있다. `Environment=prod`처럼 조직의 태그 값에 따라 필터링하는 대시보드는 리소스 태그 보강이 필요하다. [CloudWatch PromQL 조회 문서](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-PromQL-Querying.html)

## 보안 관점

보강된 지표에는 계정 ID와 리소스 ARN이 포함된다. 여러 팀이 대시보드를 공유한다면 모니터링 역할과 공유 범위가 해당 계정의 운영 정책에 맞는지 확인한다. 리소스 태그 보강을 사용한다면 태그 값도 조회할 수 있으므로 태그에 민감한 정보를 넣지 않는다.

## 운영 관점

- CloudWatch Omni를 설정하면 기본 보강이 활성화된다. 다른 계정에서도 CloudWatch 콘솔, AWS CLI 또는 AWS SDK로 활성화할 수 있다. 적용 전에 대상 리전의 CloudWatch OpenTelemetry 지표 지원 여부를 확인한다. [AWS 공식 발표](https://aws.amazon.com/about-aws/whats-new/2026/11/amazon-cloudwatch-opentelemetry-enrichment/)
- 전환 전에는 같은 인스턴스와 시간 구간을 기준으로 기존 지표 조회와 PromQL 조회의 집계 결과를 비교한다.
- 태그 기반 조회가 필요한 대시보드를 파악하고 유료 태그 보강 사용 범위를 결정한다.

## 시험 관점

CloudWatch 차원은 지표의 시계열을 구분하는 속성이다. OpenTelemetry의 리소스 속성은 그 지표가 어떤 리소스에서 나왔는지 설명하고, PromQL은 시계열을 필터링·집계하는 쿼리 언어다.

## 같이 보면 좋은 학습 포인트

- EC2 지표에서 인스턴스 차원과 리전·계정 리소스 속성의 역할을 비교한다. [CloudWatch OpenTelemetry 보강 문서](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-OTelEnrichment.html)
- 동일한 지표를 인스턴스별로 조회한 뒤 계정이나 리전 단위로 집계해 본다. [CloudWatch PromQL 조회 문서](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-PromQL-Querying.html)

## 한 줄 정리

기본 OpenTelemetry 보강을 활성화하면 지원되는 AWS 서비스 지표를 기존 CloudWatch API와 PromQL에서 조회할 수 있다.
