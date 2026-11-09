# AWS Nitro Sandboxes, EKS·ECS·EC2 워크로드에 하드웨어 격리 제공

## 공식 출처

- [Introducing AWS Nitro Sandboxes on Amazon EKS, ECS and EC2 for hardware isolated pods, tasks and VMs](https://aws.amazon.com/about-aws/whats-new/2026/11/nitro-sandboxes-eks-ecs-ec2/)
- 발표일: 2026년 11월 9일

## 핵심 내용

AWS가 Amazon EKS Pod, Amazon ECS Task 및 EC2의 경량 가상 머신을 위한 AWS Nitro Sandboxes를 출시했다. Nitro Sandboxes는 Nitro Hypervisor가 관리하는 경량 VM으로, 각 워크로드에 Nitro 기반 가상 머신의 하드웨어 격리 경계를 제공한다.

주요 특징은 다음과 같다.

- EKS Pod 또는 ECS Task마다 별도의 하드웨어 격리 VM을 제공한다.
- 신뢰할 수 없는 코드, 멀티 테넌트 워크로드, 규제 대상 워크로드를 하나의 EC2 인스턴스 안에서 분리해 실행할 수 있다.
- 암호학적 증명(cryptographic attestation)으로 샌드박스 무결성을 검증한 뒤에만 비밀 정보를 전달하도록 구성할 수 있다.
- Pod Identity, VPC CNI, EBS·EFS·S3 스토리지, CloudWatch 등 기존 AWS 통합을 그대로 사용할 수 있다.
- EKS와 ECS에서는 한 줄의 구성 변경으로 활성화할 수 있으며, EC2에서는 자체 AMI와 오케스트레이션, VMM을 이용해 직접 제어할 수 있다.
- Amazon Linux와 Firecracker를 기본 지원한다.
- 대부분의 Nitro 기반 EC2 인스턴스와 모든 상용 AWS 리전에서 추가 요금 없이 사용할 수 있다.

## 왜 중요한가

컨테이너는 일반적으로 호스트 운영체제 커널을 공유한다. 따라서 프로세스와 네임스페이스 수준의 격리는 효율적이지만, 신뢰할 수 없는 코드나 여러 고객의 워크로드를 실행할 때는 커널 공유 자체가 위험 경계를 넓힐 수 있다.

Nitro Sandboxes는 컨테이너의 운영 편의성과 VM 수준의 하드웨어 격리를 결합한다. 특히 LLM이 생성한 코드를 실행하는 AI 에이전트, SaaS의 고객별 플러그인, 코드 채점 시스템, CI 작업처럼 입력 코드의 신뢰 수준이 낮은 환경에서 유용하다. 워크로드마다 별도 EC2 인스턴스를 배정하지 않고도 하나의 인스턴스를 안전하게 공유할 수 있어 격리와 자원 활용률 사이의 균형을 개선한다.

## 서비스 및 아키텍처 관점

EKS에서는 특정 Pod가 Nitro Sandbox 안에서 실행되고, ECS에서는 특정 Task가 자체 격리 VM에서 실행된다. 애플리케이션은 기존 Kubernetes 또는 ECS 배포 모델을 유지하면서 워크로드 단위의 더 강한 격리 경계를 얻는다.

일반적인 실행 흐름은 다음과 같다.

1. EKS Pod나 ECS Task가 스케줄링된다.
2. Nitro Hypervisor가 해당 워크로드용 경량 VM을 생성한다.
3. 워크로드가 별도의 하드웨어 격리 경계 안에서 실행된다.
4. 필요하면 attestation 결과로 실행 환경의 무결성을 검증한다.
5. 검증에 성공한 환경에만 KMS 키, API 자격 증명 또는 기타 비밀 정보를 제공한다.
6. 로그와 지표는 CloudWatch 등 기존 운영 도구로 수집한다.

모든 워크로드를 샌드박스로 옮길 필요는 없다. 신뢰 수준, 규제 요구사항, 테넌트 간 격리 필요성, 시작 지연과 자원 사용량을 기준으로 적용 대상을 정하는 것이 좋다. 네트워크 정책, IAM 권한, 데이터 암호화와 이미지 공급망 보안도 별도로 설계해야 한다.

## 보안 관점

- 하드웨어 격리는 워크로드 간 커널 공유 위험과 탈출 공격의 영향을 줄이는 데 도움을 준다.
- attestation을 이용하면 승인된 이미지와 구성으로 실행된 샌드박스에만 비밀 정보를 공개할 수 있다.
- Pod Identity 또는 ECS Task Role에는 최소 권한을 적용하고, 샌드박스 자체를 신뢰의 근거로 과대평가하지 않는다.
- VPC 보안 그룹, NetworkPolicy, VPC CNI 설정으로 네트워크 접근 범위를 제한한다.
- 컨테이너 이미지 서명, 취약점 검사, 고정된 이미지 다이제스트를 함께 사용한다.
- 비밀 정보는 이미지나 환경 변수에 장기 저장하지 말고 검증된 실행 환경에서 필요할 때 가져오도록 설계한다.
- CloudTrail과 CloudWatch 로그를 중앙 수집해 샌드박스 생성, 권한 사용과 비정상 실행을 감사한다.

Nitro Sandboxes는 강한 실행 격리를 제공하지만 취약한 애플리케이션 코드, 과도한 IAM 권한, 공개된 네트워크 경로까지 자동으로 해결하지는 않는다. 심층 방어의 한 계층으로 사용해야 한다.

## 운영 관점

운영 시에는 다음 항목을 확인한다.

- 사용 중인 EC2 인스턴스가 Nitro Sandboxes를 지원하는지 여부
- EKS Pod와 ECS Task 가운데 샌드박스가 필요한 워크로드의 분류 기준
- VM 생성 시간과 애플리케이션 시작 지연
- 인스턴스별 CPU, 메모리와 샌드박스 밀도
- attestation 실패율과 비밀 정보 전달 실패
- Pod Identity, VPC CNI, 스토리지 및 CloudWatch 통합 상태
- 이미지와 Amazon Linux·Firecracker 구성 요소의 패치 수준
- 장애 시 일반 노드로 자동 우회해 보안 경계를 낮추지 않는 배포 정책

추가 서비스 요금은 없지만 기반 EC2 인스턴스와 연결된 스토리지, 네트워크 및 관측 서비스 비용은 계속 발생한다. 도입 전에는 실제 워크로드로 성능, 시작 시간과 인스턴스 집적도를 측정해야 한다.

## 시험 관점 정리

AWS 시험이나 아키텍처 문제에서는 다음 개념을 연결해서 기억할 수 있다.

- AWS Nitro System은 EC2의 가상화, 성능과 격리를 뒷받침하는 기반 기술이다.
- Nitro Sandboxes는 EKS Pod, ECS Task 또는 EC2 경량 VM에 워크로드별 하드웨어 격리를 제공한다.
- 컨테이너 격리보다 강한 경계가 필요하지만 전용 인스턴스의 비용과 낮은 활용률을 피하려는 경우에 적합하다.
- cryptographic attestation은 실행 환경의 무결성을 증명하고 비밀 정보 공개 조건으로 사용할 수 있다.
- 격리 기능과 IAM 최소 권한, 네트워크 분리, 암호화, 로깅은 서로 대체 관계가 아니라 함께 적용하는 통제다.
- EKS Pod Identity와 ECS Task Role은 워크로드별 AWS 권한을 제공한다.
- 대부분의 Nitro 기반 EC2와 모든 상용 리전에서 추가 서비스 요금 없이 제공된다.

## 같이 보면 좋은 학습 포인트

- AWS Nitro System과 Nitro Enclaves
- 컨테이너 격리와 가상 머신 격리의 차이
- Firecracker 마이크로VM 구조
- EKS Pod Identity와 ECS Task Role
- 원격 증명(remote attestation)과 신뢰 체인
- KMS를 이용한 조건부 비밀 정보 제공
- Kubernetes NetworkPolicy와 VPC CNI
- 멀티 테넌트 및 AI 에이전트 실행 환경의 위협 모델링

## 한 줄 정리

AWS Nitro Sandboxes는 EKS Pod, ECS Task와 EC2 워크로드마다 Nitro 기반 하드웨어 격리 및 무결성 증명을 제공해 신뢰할 수 없는 코드를 더 안전하고 효율적으로 실행하게 한다.
