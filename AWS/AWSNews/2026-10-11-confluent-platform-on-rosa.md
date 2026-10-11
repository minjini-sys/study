# Confluent Platform을 Red Hat OpenShift Service on AWS에서 운영하는 방법

- 정리 날짜: 2026-10-11
- 원문 게시일: 2026-10-10
- 공식 출처: [AWS IBM & Red Hat Blog - Confluent Platform on Red Hat OpenShift Service on AWS (ROSA)](https://aws.amazon.com/blogs/ibm-redhat/confluent-platform-on-red-hat-openshift-service-on-aws-rosa/)

## 핵심 내용

AWS와 Red Hat이 Red Hat OpenShift Service on AWS(ROSA)에 Confluent Platform을 배포해 enterprise event streaming 환경을 운영하는 아키텍처를 소개했다.

ROSA는 Red Hat과 AWS가 공동으로 관리하는 OpenShift 서비스다. 사용자는 control plane과 cluster lifecycle을 직접 운영하는 대신 application과 event-driven business logic에 집중할 수 있다. Confluent Platform은 Apache Kafka를 중심으로 topic, schema, connector와 stream processing 기능을 제공한다.

Confluent for Kubernetes(CFK)는 Confluent Platform 구성 요소를 Kubernetes custom resource로 관리하는 certified OpenShift operator다. 운영자는 broker, controller, certificate, upgrade와 scaling을 수동으로 구성하는 대신 원하는 상태를 선언하고 CFK가 배포, rack awareness와 recovery를 처리하도록 할 수 있다.

## 왜 중요한가

대규모 실시간 데이터 플랫폼은 Kafka cluster 자체뿐 아니라 Kubernetes cluster, control plane, certificate, network, storage, upgrade와 장애 복구를 함께 운영해야 한다. 모든 계층을 직접 관리하면 platform team의 운영 부담이 커지고 application team이 실제 event-driven 기능 개발에 집중하기 어렵다.

ROSA와 CFK를 결합하면 기반 OpenShift platform은 AWS와 Red Hat이 관리하고, Confluent lifecycle은 operator가 declarative하게 자동화한다. 조직은 기존 OpenShift governance와 GitOps 표준을 유지하면서 AWS 안에 production streaming platform을 구성할 수 있다.

특히 data residency, restricted network, deployment topology와 security policy를 직접 통제해야 해 Confluent Cloud보다 self-managed Confluent Platform이 적합한 조직에 의미가 있다.

## 주요 구성 요소

### ROSA with Hosted Control Planes

ROSA의 기본 아키텍처인 Hosted Control Planes(HCP)는 Red Hat이 control plane을 별도로 운영한다. 고객 계정에는 workload를 실행하는 worker node가 중심이 되므로 dedicated control plane EC2 instance를 직접 관리하거나 비용을 부담할 필요가 없다.

Red Hat build of Karpenter는 pod 요구 사항에 따라 worker compute를 확장하고 사용률이 낮은 node를 통합한다. fault-tolerant workload에는 Spot Instance를 활용하고 회수 상황에 대응하는 fallback을 설계할 수 있다.

### Confluent for Kubernetes

CFK는 Confluent Platform을 Kubernetes-native 방식으로 관리한다.

- Kubernetes CRD로 Confluent component의 desired state를 선언한다.
- broker와 controller의 배포 및 scaling을 자동화한다.
- rack awareness를 사용해 replica를 failure domain에 분산한다.
- upgrade와 recovery 작업을 operator workflow로 표준화한다.
- OpenShift OperatorHub와 Operator Lifecycle Manager를 통해 operator lifecycle을 관리한다.
- GitOps와 Infrastructure as Code pipeline에 streaming infrastructure를 포함할 수 있다.

## 서비스 및 아키텍처 관점

1. ROSA HCP가 managed OpenShift control plane과 AWS 기반 worker infrastructure를 제공한다.
2. CFK operator가 Kubernetes API를 통해 Confluent Platform의 desired state를 감시한다.
3. CFK가 Kafka broker, controller 및 주변 component를 OpenShift workload로 배포한다.
4. OpenShift Route 또는 private network path가 producer와 consumer의 접근 경로를 제공한다.
5. AWS PrivateLink와 private ingress를 사용해 public internet 노출 없이 service를 연결할 수 있다.
6. AWS KMS가 storage와 application data encryption에 사용할 key를 관리한다.
7. IAM role과 OpenShift access control이 AWS resource 및 cluster resource 권한을 분리한다.
8. Karpenter가 workload 요구 사항에 맞춰 EC2 worker node를 확장하거나 통합한다.

이 구조의 핵심은 Kafka를 별도 운영 영역으로 두지 않고 조직의 OpenShift platform 표준 안에 포함하는 것이다. application container와 event stream이 identity, networking, encryption, audit와 change management 기준을 공유하므로 hybrid 및 multi-cloud OpenShift 환경에서도 운영 방식을 일관되게 유지할 수 있다.

## 보안 및 운영 관점

- OpenShift Security Context Constraints(SCC)와 CFK component의 실행 요구 사항을 함께 검토한다.
- AWS managed policy와 workload용 IAM role은 least privilege 원칙으로 분리한다.
- regulated environment에서는 ROSA Customer-Approved Access를 검토해 Red Hat SRE의 support access를 승인 기반으로 통제한다.
- AWS PrivateLink, private subnet과 제한된 ingress를 사용해 streaming endpoint의 network exposure를 줄인다.
- AWS KMS key policy와 rotation policy를 정하고 data at rest encryption을 적용한다.
- client와 broker 사이의 TLS, certificate 발급, 갱신과 trust chain을 자동화한다.
- Kafka replica와 OpenShift node를 여러 Availability Zone에 분산하고 rack awareness가 실제 failure domain과 일치하는지 확인한다.
- GitOps repository의 manifest 변경에는 review와 approval을 적용하고 secret을 평문으로 저장하지 않는다.
- broker lag, under-replicated partition, disk 사용량, producer/consumer throughput과 node 상태를 함께 관찰한다.
- Spot Instance는 broker처럼 stateful하고 interruption에 민감한 workload보다 fault-tolerant workload에 신중하게 적용한다.
- upgrade 전에 compatibility, partition leadership 이동, recovery time과 rollback 절차를 검증한다.

## 비용 관점

ROSA HCP는 고객이 dedicated control plane node를 직접 운영하는 부담을 줄인다. Karpenter는 pod에 맞는 instance type을 선택하고 유휴 node를 통합해 worker compute 낭비를 줄일 수 있다.

다만 Confluent Platform은 stateful workload이므로 비용만 보고 과도하게 node를 축소하면 partition recovery와 throughput에 영향을 줄 수 있다. EC2, EBS, cross-AZ data transfer, ROSA service와 Confluent license 비용을 함께 산정하고 performance 및 resilience requirement를 기준으로 sizing해야 한다.

## 적용하기 적합한 경우

- 조직이 OpenShift를 표준 application platform으로 사용하고 있다.
- event streaming tier가 고객 AWS account와 OpenShift governance 안에 있어야 한다.
- data residency 또는 sovereignty 요구 사항이 있다.
- restricted network와 세밀한 ingress/egress 통제가 필요하다.
- Kafka deployment topology, storage와 network를 직접 제어해야 한다.
- 금융, 의료, 통신처럼 audit와 규정 준수가 중요한 실시간 workload를 운영한다.
- hybrid 또는 multi-cloud OpenShift 환경에서 같은 operator와 GitOps 운영 방식을 유지하려 한다.

완전 관리형 streaming을 빠르게 사용하려는 경우에는 Confluent Cloud와 Amazon MSK도 함께 비교해야 한다. ROSA 기반 Confluent Platform은 더 많은 통제권을 제공하지만 workload sizing, policy와 day-2 운영에 대한 고객 책임도 남는다.

## 시험 관점 정리

- ROSA는 AWS와 Red Hat이 공동으로 운영하는 managed OpenShift service다.
- Hosted Control Planes는 control plane을 고객 worker cluster와 분리해 Red Hat이 관리한다.
- Kubernetes Operator는 custom resource의 desired state를 실제 resource 상태와 지속적으로 일치시킨다.
- Karpenter는 pod scheduling 요구 사항을 기반으로 적합한 EC2 capacity를 provisioning한다.
- Kafka는 partition과 replica를 사용해 throughput과 fault tolerance를 제공한다.
- rack awareness는 replica를 서로 다른 failure domain에 배치해 단일 node 또는 AZ 장애 영향을 줄인다.
- AWS PrivateLink는 public internet을 통하지 않는 private service connectivity를 제공한다.
- AWS KMS는 encryption key의 생성, 권한, rotation과 audit를 중앙에서 관리한다.
- GitOps는 version-controlled manifest와 reconciliation을 이용해 infrastructure 및 application 변경을 관리한다.
- managed service를 사용해도 data classification, IAM, network policy와 workload configuration 책임은 고객에게 남는다.

## 같이 보면 좋은 학습 포인트

- ROSA Classic과 ROSA with Hosted Control Planes 비교
- OpenShift와 Amazon EKS의 운영 모델 차이
- Kubernetes Operator, CRD와 reconciliation loop
- Apache Kafka partition, replication과 consumer group
- Kafka rack awareness와 multi-AZ 설계
- Karpenter의 node provisioning 및 consolidation
- OpenShift Security Context Constraints
- AWS PrivateLink와 OpenShift Route
- AWS KMS envelope encryption
- Confluent Platform, Confluent Cloud와 Amazon MSK 비교

## 한 줄 정리

Confluent Platform on ROSA는 관리형 OpenShift, CFK operator와 AWS 보안 서비스를 결합해 조직의 OpenShift governance 안에서 실시간 event streaming을 declarative하고 일관되게 운영하는 아키텍처다.
