# SageMaker HyperPod에서 여러 팀이 GPU 클러스터를 안전하게 공유하는 방법

- 정리 날짜: 2026-10-09
- 원문 게시일: 2026-10-08
- 공식 출처: [AWS Artificial Intelligence Blog - Share GPU clusters across teams with isolation and fairness using Amazon SageMaker HyperPod](https://aws.amazon.com/blogs/machine-learning/share-gpu-clusters-across-teams-with-isolation-and-fairness-using-amazon-sagemaker-hyperpod/)

## 핵심 내용

AWS가 하나의 Amazon SageMaker HyperPod EKS 클러스터를 여러 AI/ML 팀이 공유하면서 identity, workload, storage와 비용을 분리하는 multi-tenant reference architecture를 공개했다.

이 구조는 AWS IAM Identity Center로 사용자를 인증하고, 팀별 SageMaker AI domain과 IAM role을 제공한다. Amazon EKS access entry와 Kubernetes RBAC는 각 팀의 접근을 전용 namespace로 제한한다. HyperPod Task Governance는 팀별 GPU quota와 workload priority를 관리하고, Kubecost는 namespace별 사용 비용을 계산한다.

각 팀은 독립된 namespace 안에서 JupyterLab 같은 HyperPod Space, 분산 학습용 HyperPod PyTorch job과 model inference endpoint를 실행한다. 고가의 GPU 인프라는 공유하지만 다른 팀의 workload와 data에는 접근하지 못하도록 경계를 둔다.

## 왜 중요한가

GPU 클러스터를 팀마다 따로 만들면 유휴 자원이 늘고 비용이 커진다. 반대로 하나의 클러스터를 단순 공유하면 특정 팀이 GPU를 독점하거나 다른 팀의 workload와 storage에 접근하고, 실제 비용을 어느 조직에 배분해야 하는지 알기 어려워진다.

multi-tenant 구조는 GPU utilization을 높이면서도 isolation, 공정한 scheduling과 chargeback을 제공한다. training, interactive development와 inference처럼 특성이 다른 workload가 같은 cluster에서 운영될 때도 quota와 priority를 명시해 중요 서비스의 자원을 보호할 수 있다.

## 아키텍처 구성

### 인증과 AWS 권한

- 기업 identity provider의 사용자와 group을 IAM Identity Center에 연결한다.
- 각 팀에 별도의 Identity Center permission set을 제공한다.
- CLI 사용자는 `aws sso login`으로 temporary credential을 받고 `kubectl`을 사용한다.
- Studio 사용자는 팀 전용 SageMaker AI domain에 로그인하고 team execution role을 사용한다.
- IAM role은 팀별 S3 prefix, CloudWatch와 필요한 SageMaker 및 EKS API로 권한을 제한한다.

### Kubernetes 접근과 workload 격리

- EKS access entry가 Studio execution role 및 CLI permission-set role을 Kubernetes 권한에 연결한다.
- Kubernetes RBAC policy를 팀 namespace 범위로 제한한다.
- 팀마다 독립된 namespace를 생성하고 Space, training job과 inference endpoint를 그 안에 배치한다.
- namespace가 다르면 다른 팀의 resource를 조회, 수정하거나 삭제할 수 없다.

### 자원 공정성과 우선순위

- HyperPod Task Governance가 namespace별 compute quota와 scheduling priority를 관리한다.
- Kueue local queue를 사용해 workload를 팀별 queue로 보낸다.
- 최소 보장량과 남는 자원의 burst 사용을 조합할 수 있다.
- 운영 inference endpoint에는 batch training보다 높은 priority를 부여해 서비스 중단을 줄일 수 있다.
- quota를 모두 사용한 팀의 job은 자원이 생길 때까지 대기하거나 policy에 따라 낮은 priority workload를 preempt한다.

### Storage 격리

- Amazon FSx for Lustre 또는 FSx for OpenZFS에 팀별 shared directory와 사용자별 home directory를 구성한다.
- 각 namespace의 PersistentVolumeClaim이 해당 팀의 directory만 mount한다.
- Amazon S3 bucket 또는 prefix 접근은 team execution role로 제한한다.
- 공유 Space를 사용할 때는 POSIX permission과 supplemental group도 함께 구성한다.

## 서비스 및 아키텍처 관점

1. IAM Identity Center가 외부 IdP와 federation하고 team membership을 관리한다.
2. 사용자는 CLI 또는 팀별 SageMaker Studio에서 HyperPod EKS cluster에 접근한다.
3. EKS access entry와 RBAC가 IAM principal을 namespace 권한에 mapping한다.
4. HyperPod Space, Training Operator와 Inference Operator가 각 namespace에서 workload를 실행한다.
5. Task Governance가 GPU quota, queue와 workload priority를 적용한다.
6. FSx 및 S3가 team-scoped training data, checkpoint와 model artifact를 저장한다.
7. HyperPod Observability와 Amazon Managed Grafana가 cluster 및 workload 지표를 제공한다.
8. Kubecost가 namespace별 GPU, CPU, memory, storage와 network 비용을 집계한다.

이 설계는 identity부터 IAM, Kubernetes authorization, compute scheduling, storage와 cost reporting까지 같은 team boundary를 일관되게 사용한다. 한 계층에서만 분리하면 다른 경로를 통한 cross-team access나 비용 혼합이 남을 수 있다.

## 보안 및 운영 관점

- 외부 IdP의 group membership을 source of truth로 사용하고 SCIM으로 Identity Center에 동기화한다.
- Studio execution role과 CLI permission-set role을 분리해 각 접근 경로의 권한을 독립적으로 제한한다.
- EKS access entry는 반드시 team namespace 범위의 RBAC policy와 연결한다.
- Pod에서 AWS resource에 접근할 때는 EKS Pod Identity와 team-scoped IAM role을 사용한다.
- S3 bucket 및 prefix, KMS key와 FSx directory의 권한도 namespace 경계와 일치시킨다.
- cluster administrator와 platform service account는 일반 team role과 분리한다.
- admission policy와 resource limit으로 privileged container, host path와 과도한 resource request를 제한한다.
- inference에는 높은 priority를 주되 preemption이 training checkpoint와 데이터 일관성에 미치는 영향을 검토한다.
- Grafana team 사용자는 Viewer role로 제한하고 namespace별 dashboard visibility를 설정한다.
- Kubecost budget 및 alert를 namespace별로 만들어 quota와 실제 비용을 함께 관찰한다.

## 비용 및 운영 가시성

Kubecost는 Kubernetes namespace, label, deployment와 service를 기준으로 cluster 비용을 분해한다. 팀마다 namespace 하나를 사용하면 별도 tagging 없이 GPU, CPU, memory, storage와 network 사용량을 team 비용으로 연결할 수 있다.

platform team은 namespace별 budget 및 alert를 설정하고 chargeback 또는 showback report를 만들 수 있다. Task Governance의 quota는 사용할 수 있는 자원을 통제하고 Kubecost는 실제 소비 비용을 보여주므로 두 기능을 함께 사용해야 한다.

## 시험 관점 정리

- SageMaker HyperPod는 대규모 AI training 및 inference cluster의 health monitoring, fault recovery와 lifecycle을 관리한다.
- HyperPod는 Amazon EKS 또는 Slurm orchestration을 지원한다.
- IAM Identity Center permission set은 사용자에게 AWS account 접근용 temporary role을 제공한다.
- EKS access entry는 IAM principal을 Kubernetes 접근 권한에 연결한다.
- Kubernetes namespace와 RBAC는 logical isolation을 제공하지만 node-level isolation과 network policy는 별도로 검토해야 한다.
- quota는 자원 사용 상한 또는 보장량을 정하고 priority는 자원이 부족할 때 scheduling 순서를 결정한다.
- FSx for Lustre는 distributed training에 필요한 고성능 POSIX file system으로 사용할 수 있다.
- showback은 비용을 보여주고 chargeback은 실제로 조직에 비용을 배분한다.

## 같이 보면 좋은 학습 포인트

- SageMaker HyperPod EKS와 Slurm 비교
- EKS access entry와 Kubernetes RBAC
- IAM Identity Center, SAML 2.0 및 SCIM
- EKS Pod Identity와 IRSA 비교
- Kubernetes namespace, quota와 network policy
- Kueue local queue 및 workload priority
- distributed training checkpoint 전략
- FSx for Lustre와 FSx for OpenZFS
- Kubecost 기반 GPU chargeback

## 한 줄 정리

SageMaker HyperPod multi-tenant architecture는 IAM Identity Center, team별 namespace, Task Governance와 Kubecost를 결합해 하나의 GPU cluster를 격리, 공정성 및 비용 가시성을 유지하며 여러 팀이 공유하게 한다.
