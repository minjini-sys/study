# Amazon EC2 Capacity Manager, 지원 가능성 점검과 용량 충족 지표 추가

## 공식 출처

- [New features for Amazon EC2 Capacity Manager](https://aws.amazon.com/about-aws/whats-new/2026/11/ec2-capacity-manager-supportability-and-fulfillment/)
- [Plan and measure EC2 flexibility with Amazon EC2 Capacity Manager](https://aws.amazon.com/blogs/compute/plan-and-measure-ec2-flexibility-with-amazon-ec2-capacity-manager/)
- 발표일: 2026년 11월 4일

## 핵심 내용

Amazon EC2 Capacity Manager에 실시간 지원 가능성 점검(real-time supportability checks)과 용량 충족 지표(capacity fulfillment metrics)가 추가되었다.

실시간 지원 가능성 점검은 인스턴스를 실제로 실행하거나 예약하지 않고도 요청한 용량을 각 인스턴스 유형과 가용 영역 조합이 얼마나 잘 지원할 수 있는지 평가한다. 각 조합과 전체 구성에 `High`, `Medium`, `Low` 점수를 제공한다.

용량 충족 지표는 실제 EC2 시작 요청을 기준으로 요청한 용량과 제공된 용량을 비교한다. AWS Organizations, 계정, 리전, 태그가 지정된 워크로드 단위로 확인하고 인스턴스 유형과 가용 영역별로 분석할 수 있다.

두 기능은 대규모 미션 크리티컬 워크로드를 위한 AWS Support 플랜인 AWS Unified Operations에 포함되며 모든 상용 AWS 리전에서 사용할 수 있다.

## 왜 중요한가

대규모 확장 시 특정 인스턴스 유형과 단일 가용 영역만 사용하면 그 조합의 가용 용량이 부족할 때 시작 요청이 실패할 가능성이 높아진다. 여러 인스턴스 유형과 가용 영역을 허용하면 EC2가 요청을 충족할 선택지가 늘어난다.

Capacity Manager의 새 기능은 용량 계획의 두 시점을 연결한다.

- 확장 전: 어떤 인스턴스 유형과 가용 영역 조합이 요청을 지원할 가능성이 높은지 확인
- 확장 후: 실제 요청한 vCPU 중 얼마나 제공되었는지 측정

이를 통해 단순히 구성을 유연하게 만들었다고 가정하지 않고, 유연성이 실제 용량 확보에 효과가 있었는지 데이터로 검증할 수 있다.

## 서비스 및 아키텍처 관점

지원 가능성 점검에는 On-Demand 또는 Spot 시장 유형, 필요한 용량, 후보 가용 영역과 인스턴스 유형을 입력한다. Capacity Manager는 각 `인스턴스 유형 x 가용 영역` 조합과 전체 후보 집합을 평가한다.

점수의 의미는 다음과 같다.

- High: 요청을 충족하고도 여유가 있는 수준
- Medium: 요청을 충족할 수 있지만 여유가 적은 수준
- Low: 요청한 용량보다 지원 가능한 용량이 적은 수준

점수는 현재 상황을 바탕으로 한 계획 지침이지 용량 제공 약속이 아니다. 특정 유형과 가용 영역에 확정적인 용량이 필요하면 On-Demand Capacity Reservation 또는 future-dated Capacity Reservation을 검토해야 한다.

용량 충족 지표에는 다음 항목이 포함된다.

- `CapacityTotalRequestedVcpu`: 요청한 총 vCPU
- `CapacityTotalFulfilledVcpu`: 실제 제공된 총 vCPU
- `CapacityRequestFulfillmentRate`: 요청 대비 충족 비율을 vCPU 기준으로 계산
- `AvgInstAzCountPerRequestedVcpu`: 요청한 vCPU당 사용할 수 있도록 구성된 인스턴스 유형과 가용 영역 조합의 평균 수

일반적인 개선 흐름은 다음과 같다.

1. 조직, 계정, 리전 또는 워크로드 태그별 충족률을 확인한다.
2. 목표보다 낮은 워크로드를 인스턴스 유형과 가용 영역별로 분석한다.
3. 대규모 확장 전에 후보 조합의 지원 가능성 점검을 실행한다.
4. 점수가 높은 후보를 Auto Scaling 그룹, EC2 Fleet 또는 Karpenter 구성에 추가한다.
5. 필요한 최소 기준 용량만 Capacity Reservation으로 확보한다.
6. 변경 후 충족률과 조합 수 지표를 다시 확인한다.

## 보안 관점

- Capacity Manager 조회와 설정 권한은 플랫폼 및 용량 계획 담당 역할에 최소 권한으로 부여한다.
- 조직 수준 활성화는 관리 계정 또는 승인된 위임 관리자에서 수행한다.
- 계정, 리전, 워크로드 태그별 용량 데이터는 내부 인프라 규모를 드러낼 수 있으므로 접근 대상을 제한한다.
- 모니터링 태그 키를 표준화하고 민감한 정보를 태그 값에 포함하지 않는다.
- CloudTrail을 통해 Capacity Manager API와 Capacity Reservation 변경 이력을 감사한다.
- 자동화가 점수만 보고 과도한 권한으로 인스턴스나 예약을 생성하지 않도록 승인과 비용 통제를 둔다.

용량 가시성은 보안 통제를 대체하지 않는다. 후보 인스턴스 유형을 늘릴 때도 아키텍처, 운영체제, 라이선스와 규정 준수 요구사항을 충족하는 유형만 허용해야 한다.

## 운영 관점

지원 가능성 점검은 워크로드 계획이나 대규모 변경 전에 실행하는 것이 적합하다. 매 시작 요청마다 호출하면 불필요한 리전 간 의존성과 시간당 할당량 사용이 생길 수 있다.

운영 시에는 다음 항목을 관리한다.

- On-Demand와 Spot의 충족률을 분리해 확인
- 워크로드별 요청 vCPU와 충족 vCPU
- 인스턴스 유형 및 가용 영역 조합 수
- 낮은 충족률이 반복되는 유형과 가용 영역
- Auto Scaling, EC2 Fleet, Karpenter의 유연성 구성
- 지원 가능성 점검 시점과 결과의 유효성
- Capacity Reservation의 사용률과 비용
- 조직 및 계정별 태그 품질과 모니터링 범위

Capacity Reservation은 사용하지 않아도 비용이 발생할 수 있으므로 정상 상태에서 반드시 필요한 기준 용량에 맞추고, 나머지는 여러 인스턴스 유형과 가용 영역을 이용한 유연성으로 확보하는 방식이 합리적이다.

## 시험 관점 정리

AWS 시험이나 아키텍처 문제에서는 다음 개념을 연결해서 기억할 수 있다.

- 여러 인스턴스 유형과 가용 영역을 허용하면 EC2 시작 요청의 충족 가능성을 높일 수 있다.
- 지원 가능성 점검은 인스턴스를 실행하거나 예약하지 않고 후보 조합을 평가한다.
- High, Medium, Low 점수는 계획 지침이며 용량 보장이 아니다.
- 확정 용량이 필요하면 EC2 Capacity Reservation을 사용한다.
- 용량 충족률은 요청한 vCPU 대비 실제 제공된 vCPU를 보여준다.
- Auto Scaling의 Mixed Instances Policy와 EC2 Fleet는 인스턴스 유형 유연성을 제공한다.
- Karpenter는 요구사항을 충족하는 여러 인스턴스 유형에서 노드를 선택할 수 있다.
- Spot은 가격 효율성이 높지만 중단 가능성과 용량 변동을 고려해야 한다.

## 같이 보면 좋은 학습 포인트

- EC2 On-Demand Capacity Reservation
- future-dated Capacity Reservation
- Auto Scaling Mixed Instances Policy
- EC2 Fleet의 할당 전략
- Karpenter의 NodePool과 인스턴스 요구사항
- Attribute-based instance type selection
- Spot Instance의 용량 최적화 전략
- AWS Organizations의 위임 관리자와 태그 거버넌스

## 한 줄 정리

EC2 Capacity Manager의 지원 가능성 점검과 용량 충족 지표를 이용하면 대규모 확장 전에 유리한 인스턴스 유형·가용 영역 조합을 찾고, 확장 후에는 유연성이 실제 용량 확보에 기여했는지 측정할 수 있다.
