# GuardDuty Extended Threat Detection에 공격 시퀀스 지표 5가지 추가

- 정리 날짜: 2026-10-14
- 원문 게시일: 2026-10-12
- 공식 출처: [AWS Security Blog - GuardDuty Extended Threat Detection has added five new indicators to surface attack sequences](https://aws.amazon.com/blogs/security/guardduty-extended-threat-detection-has-added-five-new-indicators-to-surface-attack-sequences/)

## 핵심 내용

Amazon GuardDuty Extended Threat Detection이 여러 개의 약한 보안 신호를 연관 분석할 때 사용할 수 있는 지표 5가지를 추가했다.

- `VULNERABILITY`: Amazon Inspector가 제공하는 CVE, CVSS 심각도와 EPSS 악용 가능성
- `MALICIOUS_PACKAGE`: Amazon Inspector가 탐지한 악성 package 정보
- `REACHABILITY`: Amazon VPC Reachability Analyzer가 확인한 internet 접근 가능성
- `MISCONFIGURATION`: AWS Security Hub control이 탐지한 보안 설정 오류
- `SENSITIVE_DATA`: Amazon Macie가 탐지한 잠재적인 민감 데이터

각 정보는 단독으로는 실제 공격을 의미하지 않을 수 있다. GuardDuty는 이를 process execution, pod exec, API activity와 network indicator 같은 runtime 신호와 결합해 신뢰도가 임계값을 넘으면 하나의 `CRITICAL` attack sequence finding으로 제공한다.

새 지표는 GuardDuty console의 attack-sequence finding에 있는 Indicators tab과 `GetFindings` API의 JSON 응답에서 확인할 수 있다.

## 왜 중요한가

보안 운영팀은 취약점, 잘못된 설정과 수상한 runtime activity를 각각 다른 도구에서 확인해야 했다. 개별 경고는 맥락이 부족해 false positive처럼 보이거나 우선순위가 낮게 처리될 수 있다.

예를 들어 한 EC2 instance에 CVE가 있다는 사실만으로는 침해를 의미하지 않는다. 그러나 같은 instance가 internet에서 접근 가능하고, 악성 package가 존재하며, 수상한 process가 실행되고 있다면 실제 공격일 가능성이 크게 높아진다.

Extended Threat Detection은 activity와 resource-state context를 하나의 공격 흐름으로 묶어 SOC가 많은 경고를 따로 조사하는 시간을 줄인다. 어떤 resource를 격리하고 어떤 vulnerability를 먼저 patch해야 하는지도 finding 안에서 확인할 수 있다.

## 지표별 의미

### VULNERABILITY

Amazon Inspector의 CVE 목록과 CVSS, EPSS 정보를 제공한다. CVSS는 vulnerability 자체의 기술적 심각도를 나타내고, EPSS는 공개된 CVE가 향후 30일 안에 실제로 악용될 가능성을 0에서 1 사이로 추정한다.

incident response에서는 CVSS만 높은 항목보다 EPSS가 높고 현재 공격 시퀀스와 연결된 vulnerability를 먼저 처리하는 것이 효과적이다.

### MALICIOUS_PACKAGE

Amazon Inspector의 `MAL-*` indicator를 통해 workload에 알려진 악성 package가 존재하는지 보여준다. container image 안에 악성 binary가 포함된 경우 worker node만 교체하고 같은 image를 다시 배포하면 문제가 반복될 수 있으므로 image와 software supply chain까지 조사해야 한다.

### REACHABILITY

VPC Reachability Analyzer 정보로 host, cluster API endpoint 또는 worker node가 internet에서 접근 가능한지 보여준다. exploitable vulnerability와 public reachability가 동시에 확인되면 즉시 network isolation을 검토해야 한다.

### MISCONFIGURATION

Security Hub의 posture management 결과를 활용한다. IMDSv1 사용, 과도한 administrator policy와 public S3 bucket처럼 AWS 또는 업계 best practice에 맞지 않는 설정을 공격 맥락에 추가한다.

### SENSITIVE_DATA

Amazon Macie가 resource에서 잠재적인 민감 데이터를 발견했는지 보여준다. 침해된 workload가 민감 데이터에 접근할 수 있다면 incident의 영향 범위와 대응 우선순위가 높아진다.

## 필요한 서비스

- Amazon EC2와 Amazon EKS에 대한 Amazon Inspector
- AWS Foundational Security Best Practices control을 사용하는 AWS Security Hub
- production VPC를 분석하는 Amazon VPC Reachability Analyzer
- EC2, ECS와 EKS에 대한 GuardDuty Runtime Monitoring
- S3의 민감 데이터를 탐지하는 Amazon Macie

이 서비스들이 이미 활성화되어 있다면 해당 데이터가 새로운 attack-sequence finding에 자동으로 나타난다. 각 서비스의 coverage가 빠진 resource에서는 일부 지표가 비어 있을 수 있으므로 조직 전체의 enablement 상태를 확인해야 한다.

## 서비스 및 아키텍처 관점

1. GuardDuty Runtime Monitoring이 EC2, ECS와 EKS의 process, syscall 및 container activity를 수집한다.
2. GuardDuty가 CloudTrail, EKS audit log, DNS와 network 기반의 의심스러운 활동을 탐지한다.
3. Amazon Inspector가 workload의 vulnerability와 malicious package를 평가한다.
4. VPC Reachability Analyzer가 resource의 network exposure를 분석한다.
5. Security Hub가 configuration 및 security posture finding을 통합한다.
6. Macie가 S3 data에서 sensitive data를 식별한다.
7. Extended Threat Detection이 시간과 resource 관계를 기준으로 신호를 상관 분석한다.
8. 신뢰도 임계값을 넘은 활동을 MITRE ATT&CK 단계가 포함된 단일 attack sequence로 제공한다.

이 구조는 여러 security service를 단순히 한 화면에 모으는 것을 넘어 activity signal과 resource context를 결합한다. 강한 network IOC가 이미 있는 경우에도 vulnerability context가 추가되면 최초 침투 경로를 추정하고 근본 원인을 patch하는 데 도움이 된다.

## 보안 및 운영 관점

- `CRITICAL` attack sequence는 개별 finding보다 우선해 즉시 triage한다.
- `MALICIOUS_PACKAGE`와 높은 우선순위의 `VULNERABILITY`가 있으면 host 격리와 patch를 함께 수행한다.
- 노출 가능성이 있는 IAM user credential, instance profile과 IRSA role을 rotate한다.
- `REACHABILITY`가 표시된 public resource는 remediation 동안 network에서 격리한다.
- patch 순서는 CVSS만 사용하지 않고 EPSS, internet reachability와 실제 runtime activity를 함께 고려한다.
- container 환경에서는 node뿐 아니라 image, registry와 CI/CD pipeline까지 조사한다.
- IMDSv1이 표시되면 IMDSv2 전환과 hop limit 및 metadata option을 검토한다.
- finding의 resource timeline, source IP, identity, process와 API call을 보존해 forensic evidence로 사용한다.
- EventBridge와 Security Hub automation rule을 사용해 ticket 생성, 격리와 notification workflow를 연결할 수 있다.
- 자동 격리는 service availability에 영향을 줄 수 있으므로 resource criticality와 승인 절차를 runbook에 포함한다.
- EC2, ECS와 EKS coverage 및 Runtime Monitoring agent 상태를 지속적으로 점검한다.

## 시험 관점 정리

- Amazon GuardDuty는 AWS account와 workload의 위협을 지속적으로 탐지하는 managed threat detection service다.
- GuardDuty Extended Threat Detection은 관련 신호를 시간 순서의 attack sequence로 연결한다.
- Amazon Inspector는 EC2, ECR container image와 Lambda 등의 software vulnerability를 평가한다.
- Amazon Macie는 machine learning과 pattern matching으로 S3의 sensitive data를 탐지한다.
- AWS Security Hub는 여러 보안 서비스의 finding과 security standard 결과를 통합한다.
- VPC Reachability Analyzer는 configuration을 분석해 source와 destination 사이의 network path 가능성을 확인한다.
- CVSS는 vulnerability의 기술적 심각도이고 EPSS는 가까운 시점의 실제 악용 가능성을 추정한다.
- IMDSv2는 session-oriented token을 요구해 SSRF를 통한 metadata credential 탈취 위험을 줄인다.
- IRSA는 EKS pod의 service account에 IAM role을 연결한다.
- MITRE ATT&CK mapping은 공격자의 tactic과 technique을 공통 분류 체계로 표현한다.

## 같이 보면 좋은 학습 포인트

- GuardDuty Extended Threat Detection과 attack sequence
- GuardDuty Runtime Monitoring for EC2, ECS and EKS
- Amazon Inspector vulnerability 및 package scanning
- CVSS와 EPSS의 차이
- AWS Security Hub security standards
- Amazon Macie sensitive data discovery
- VPC Reachability Analyzer와 Network Access Analyzer
- EC2 IMDSv2와 metadata 보안
- EKS IRSA 및 Pod Identity
- EventBridge 기반 incident response automation

## 한 줄 정리

GuardDuty Extended Threat Detection은 vulnerability, malicious package, network reachability, misconfiguration과 sensitive data context를 runtime 신호와 결합해 대응 가치가 높은 공격을 하나의 `CRITICAL` attack sequence로 보여준다.
