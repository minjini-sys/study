# Amazon EKS와 EKS Distro, Kubernetes 1.37 지원

- 정리 날짜: 2026-10-03
- 원문 게시일: 2026-10-02
- 공식 출처: [AWS What's New - Amazon EKS and Amazon EKS Distro now support Kubernetes version 1.37](https://aws.amazon.com/about-aws/whats-new/2026/10/amazon-eks-distro-kubernetes-version-1-37/)

## 핵심 내용

Amazon Elastic Kubernetes Service(EKS)와 Amazon EKS Distro가 Kubernetes 1.37을 지원한다. 신규 EKS 클러스터를 1.37로 생성하거나 기존 클러스터를 EKS 콘솔, `eksctl` 또는 Infrastructure as Code 도구를 사용해 업그레이드할 수 있다.

이번 버전에서는 Metrics API가 `metrics.k8s.io/v1`으로 정식 출시되었다. Pod와 노드의 CPU 및 메모리 사용량을 Horizontal Pod Autoscaler(HPA)와 `kubectl top`에 제공하는 표준 인터페이스가 안정화된 것이다.

Dynamic Resource Allocation(DRA)의 device taint와 toleration도 정식 출시되었다. GPU 같은 개별 장치에 taint를 설정해 해당 장치를 명시적으로 허용한 워크로드만 스케줄링되도록 제어할 수 있다. HPA의 scale-to-zero 기능은 베타로 승격되고 기본 활성화되어, object 또는 external metric을 사용하는 워크로드가 유휴 상태일 때 Pod 수를 0까지 줄일 수 있다.

## 왜 중요한가

Kubernetes 업그레이드는 새로운 기능을 사용하는 문제뿐 아니라 보안 패치, API 호환성과 지원 수명에 직접 연결된다. EKS가 최신 버전을 지원하면 관리형 control plane을 유지하면서 새로운 스케줄링 및 자동 확장 기능을 사용할 수 있다.

특히 GPU와 가속기를 사용하는 AI 워크로드에서는 장치 단위 격리와 할당 정책이 중요하다. DRA device taint 및 toleration은 비싼 장치가 의도하지 않은 Pod에 배정되는 일을 줄인다. HPA scale-to-zero는 이벤트 기반 또는 간헐적 워크로드의 유휴 비용을 낮출 수 있다.

## 주요 변경 사항

### Metrics API 정식 출시

- `metrics.k8s.io/v1` API가 GA 상태가 되었다.
- Pod와 노드의 CPU 및 메모리 사용량을 제공한다.
- HPA의 리소스 지표와 `kubectl top` 명령에서 활용된다.
- 안정된 API 계약을 기반으로 모니터링 및 자동 확장 구성을 운영할 수 있다.

### DRA device taint 및 toleration 정식 출시

- DRA 드라이버 또는 관리자가 GPU 같은 특정 장치에 taint를 설정할 수 있다.
- 필요한 toleration을 가진 워크로드만 해당 장치를 사용하도록 스케줄러가 제어한다.
- 노드 전체가 아니라 개별 장치 수준의 배치 정책을 구성할 수 있다.

### HPA scale-to-zero 베타

- `minReplicas: 0`을 설정한 HPA가 유휴 시 Pod 수를 0으로 줄일 수 있다.
- object metric 또는 external metric을 사용하는 경우 적용할 수 있다.
- 수요가 다시 생기면 지표를 기반으로 Pod를 확장한다.
- 베타 기능이므로 프로덕션 적용 전 지표 가용성과 복구 시간을 검증해야 한다.

## 서비스 및 아키텍처 관점

EKS에서는 AWS가 Kubernetes control plane의 가용성과 패치를 관리하고 사용자는 worker node, add-on과 워크로드 구성을 관리한다. 버전 업그레이드는 control plane에서 시작해 managed node group 또는 자체 관리 노드, 핵심 add-on과 애플리케이션 순서로 진행하는 것이 일반적이다.

Metrics Server가 Pod 및 노드의 리소스 사용량을 Metrics API로 노출하면 HPA가 이 데이터를 바탕으로 replica 수를 조정한다. 외부 지표 기반 scale-to-zero를 사용할 때는 Amazon CloudWatch 또는 다른 지표 시스템과 adapter의 지속적인 가용성이 중요하다. 지표가 중단되면 필요한 시점에 워크로드가 확장되지 않을 수 있다.

DRA는 장치 드라이버가 제공하는 GPU 등 특수 하드웨어를 워크로드에 동적으로 할당한다. device taint와 toleration을 함께 사용하면 가속기 풀의 용도, 보안 경계와 비용 정책을 스케줄링에 반영할 수 있다.

## 보안 및 운영 관점

- EKS cluster insights로 업그레이드 전에 폐기된 API, add-on 호환성 및 잠재적인 문제를 확인한다.
- 개발 또는 staging 클러스터에서 먼저 업그레이드하고 핵심 워크로드의 동작과 rollback 절차를 검증한다.
- control plane, 노드와 add-on의 지원 버전 조합을 확인하고 단계적으로 업그레이드한다.
- PodDisruptionBudget, topology spread constraint와 여러 가용 영역 구성을 점검해 노드 교체 중 가용성을 유지한다.
- HPA scale-to-zero 적용 시 cold start, 지표 지연과 최소 처리 용량 요구 사항을 검토한다.
- Metrics API와 외부 지표 adapter에 대한 RBAC 권한을 최소화하고 불필요한 지표 접근을 제한한다.
- DRA 장치의 taint 및 toleration 정책을 코드로 관리해 GPU가 일반 워크로드에 잘못 배정되지 않도록 한다.
- 업그레이드 후 CloudWatch Container Insights, Kubernetes 이벤트와 애플리케이션 SLI를 관찰한다.
- 지원 종료 전에 다음 버전으로 이동할 수 있도록 EKS 버전 수명 주기를 운영 일정에 포함한다.

## 시험 관점 정리

- Amazon EKS는 AWS가 Kubernetes control plane을 관리하는 서비스다.
- EKS Distro는 Amazon EKS가 사용하는 Kubernetes 구성 요소의 오픈 소스 배포판이다.
- HPA는 지표를 기준으로 Deployment 또는 StatefulSet 등의 replica 수를 자동 조정한다.
- Metrics Server는 리소스 사용량을 Metrics API로 제공하지만 장기 모니터링 저장소를 대신하지 않는다.
- taint는 특정 노드나 장치를 일반 워크로드가 사용하지 못하게 하고 toleration은 해당 제한을 허용한다.
- scale-to-zero는 비용을 절감할 수 있지만 첫 요청의 지연 시간과 지표 파이프라인 가용성을 고려해야 한다.
- EKS 업그레이드 전에 API 호환성, add-on, 노드 이미지와 애플리케이션을 함께 점검해야 한다.
- managed node group은 노드 교체와 버전 업데이트 작업을 단순화한다.

## 제공 범위

Kubernetes 1.37은 EKS가 제공되는 모든 AWS 리전에서 사용할 수 있으며 AWS GovCloud(US) 리전도 포함된다. EKS Distro 1.37 빌드는 Amazon ECR Public Gallery와 GitHub에서 제공된다.

## 같이 보면 좋은 학습 포인트

- EKS 버전 수명 주기와 표준 및 연장 지원 정책
- EKS cluster insights를 이용한 업그레이드 준비
- Metrics Server와 CloudWatch Container Insights의 역할 차이
- HPA, VPA와 Cluster Autoscaler 또는 Karpenter 비교
- custom, object 및 external metric 기반 자동 확장
- Kubernetes DRA와 device plugin의 차이
- GPU 워크로드의 taint, toleration과 node affinity
- EKS managed node group의 순차 업데이트 전략
- PodDisruptionBudget과 다중 가용 영역 고가용성

## 한 줄 정리

Amazon EKS의 Kubernetes 1.37 지원은 안정화된 Metrics API, 장치 단위 DRA 제어와 HPA scale-to-zero를 통해 관측성, 가속기 스케줄링과 유휴 비용 최적화를 강화한다.
