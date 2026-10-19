# AWS에서 ML-DSA 알고리즘 정책을 강제하는 방법

- 정리 날짜: 2026-10-19
- 원문 게시일: 2026-10-12
- 공식 출처: [AWS Security Blog - Quantum-resistant signing on AWS, Part 2: Enforce your ML-DSA algorithm policy with SCPs and IAM](https://aws.amazon.com/blogs/security/quantum-resistant-signing-on-aws-part-2-enforce-your-ml-dsa-algorithm-policy-with-scps-and-iam/)

## 핵심 내용

AWS가 post-quantum cryptography에 사용하는 ML-DSA parameter set을 조직 정책에 맞게 제한하는 방법을 공개했다. 예시 정책은 AWS KMS가 지원하는 `ML_DSA_44`, `ML_DSA_65`, `ML_DSA_87` 중 `ML_DSA_65`와 `ML_DSA_87`만 승인하고 `ML_DSA_44`의 신규 생성과 기존 key 사용을 차단한다.

중요한 제어 지점은 `kms:SigningAlgorithm`이 아니라 `kms:KeySpec` condition key다. 세 parameter set 모두 AWS KMS에서 `ML_DSA_SHAKE_256` signing algorithm을 사용하므로 `kms:SigningAlgorithm`만으로는 variant를 구분할 수 없다. 반면 `kms:KeySpec`은 key 생성뿐 아니라 `Sign`, `Verify`, `GetPublicKey` 같은 resource operation에서도 평가되어 특정 parameter set을 정확히 제한할 수 있다.

AWS는 하나의 policy만 적용하는 대신 SCP, IAM permission boundary, KMS key policy, 승인된 provisioning pipeline, AWS CloudTrail과 AWS Config를 결합한 defense-in-depth 방식을 제안한다.

## 왜 중요한가

양자 내성 알고리즘을 도입하는 것만으로는 조직 전체의 암호 정책이 자동으로 지켜지지 않는다. account와 team이 늘어나면 승인되지 않은 parameter set이 새로 생성되거나 기존 key가 계속 사용될 수 있다. 따라서 알고리즘 선택을 문서에만 적는 것이 아니라 AWS control plane에서 예방하고 이탈을 탐지해야 한다.

또한 암호 표준과 조직 요구 사항은 계속 바뀐다. 오늘 승인한 알고리즘도 미래에는 교체될 수 있으므로 architecture를 다시 만들지 않고 허용 목록을 바꿀 수 있는 cryptographic agility가 필요하다.

이 글의 `ML_DSA_44` 차단은 해당 variant가 안전하지 않다는 뜻이 아니다. ML-DSA-44는 NIST FIPS 204의 표준 parameter set이며 작은 key와 signature, 빠른 처리 속도가 필요한 환경에 적합하다. 조직이 장기 신뢰성, 내부 표준 또는 규제 요구에 따라 더 큰 security margin을 가진 ML-DSA-65와 ML-DSA-87을 선택했다는 예시다.

## 주요 제어 방법

### 1. AWS Organizations SCP

organization root 또는 production OU에 SCP를 연결해 member account가 승인되지 않은 key를 만들거나 사용하지 못하게 한다.

- `kms:CreateKey`에서 `kms:KeySpec`이 `ML_DSA_44`인 요청을 거부한다.
- 기존 key도 사용할 수 없도록 `kms:Sign`, `kms:Verify`, `kms:GetPublicKey`를 거부한다.
- multi-Region key를 사용하면 다른 Region에 같은 key spec을 복제하지 못하도록 `kms:ReplicateKey`도 검토한다.
- SCP의 explicit deny는 member account의 IAM allow로 우회할 수 없다.
- SCP는 management account에는 적용되지 않으므로 management account에서 workload를 운영하지 않는 원칙이 중요하다.

`kms:*` 전체를 거부하면 `DescribeKey`, `DisableKey`, `ScheduleKeyDeletion`까지 막혀 문제가 있는 key를 조사하거나 정리하기 어려워질 수 있다. 차단 대상 action을 필요한 범위로 제한해야 한다.

### 2. IAM policy와 permission boundary

조직 전체가 아니라 특정 account, role 또는 개발 환경에 유연한 예외가 필요하면 IAM policy와 permission boundary를 사용할 수 있다.

permission boundary는 role이 받을 수 있는 최대 권한을 제한한다. 하지만 새 role에 boundary를 빠뜨리지 않도록 IaC, account vending 또는 CI/CD에서 부착을 자동화해야 한다. 강제 배포 절차가 없다면 SCP보다 쉽게 누락될 수 있다.

### 3. KMS key policy

이미 존재하는 ML-DSA key의 사용을 제한하려면 KMS key policy에서도 `kms:KeySpec`을 기준으로 `Sign`, `Verify`, `GetPublicKey`를 거부할 수 있다. SCP와 함께 사용하면 조직 범위와 resource 범위에서 중첩된 통제를 만들 수 있다.

### 4. AWS Private CA provisioning 통제

AWS Private CA의 `CreateCertificateAuthority`에는 signing algorithm을 검사하는 IAM condition key가 없다. 따라서 SCP만으로 `ML_DSA_44` CA 생성을 조건부 차단할 수 없다.

이 경우 다음 방식을 사용한다.

1. CA를 생성할 수 있는 principal을 소수의 승인된 role 또는 pipeline으로 제한한다.
2. 검증된 IaC template에 `ML_DSA_65` 또는 `ML_DSA_87`을 고정한다.
3. CSR의 subject key algorithm도 같은 provisioning pipeline에서 검증한다.
4. 승인된 CA만 trust store와 certificate chain에 포함한다.
5. CloudTrail과 AWS Config custom rule로 정책 이탈을 탐지한다.

### 5. CloudTrail과 AWS Config 탐지

예방 통제가 모든 service surface를 완전히 검사할 수는 없다. CloudTrail은 `kms:CreateKey`의 `KeySpec`과 `acm-pca:CreateCertificateAuthority`의 signing algorithm 같은 API input을 기록한다.

AWS Config custom rule은 현재 resource configuration을 평가해 승인되지 않은 key 또는 CA가 남아 있는지 찾는 detective control로 사용할 수 있다. 인증서 ARN만 참조하는 listener 변경처럼 event 자체에 알고리즘 정보가 없는 경우에는 resource 상태 평가가 특히 중요하다.

## 서비스 및 아키텍처 관점

권장 흐름은 다음과 같다.

1. 보안 및 compliance team이 승인할 ML-DSA parameter set을 정의한다.
2. AWS Organizations의 root 또는 OU에 SCP를 연결해 비승인 KMS key 생성과 사용을 차단한다.
3. account 내부의 세부 권한은 IAM permission boundary와 KMS key policy로 제한한다.
4. AWS Private CA 생성은 승인된 provisioning role과 IaC pipeline만 수행한다.
5. pipeline은 CA와 CSR의 algorithm을 검증하고 승인된 configuration을 고정한다.
6. CloudTrail organization trail이 key, CA와 certificate 관련 API event를 중앙 수집한다.
7. AWS Config custom rule과 alerting이 정책 이탈을 탐지한다.
8. 정기 review에서 새 parameter set, API action과 표준 변경을 policy에 반영한다.

중요한 점은 preventive control과 detective control을 나누는 것이다. condition key가 지원되는 KMS에는 요청 자체를 막는 preventive control을 적용하고, algorithm-aware condition이 없는 Private CA 또는 certificate import surface에는 승인된 경로와 사후 탐지를 결합한다.

## 보안 관점

- deny list는 새 알고리즘이 출시되면 기본적으로 허용될 수 있다. 미평가 variant까지 차단하려면 `StringNotEquals`로 승인 목록을 정의하는 default-deny 방식을 고려한다.
- 승인 목록 방식은 새로운 variant를 사용하기 전에 policy update가 필요하지만 unknown algorithm이 자동으로 허용되는 위험을 줄인다.
- SCP는 member account에만 적용되므로 management account를 일반 workload account로 사용하지 않는다.
- CA 생성, certificate import와 upload 권한을 최소화하고 승인된 pipeline에 집중한다.
- signing key와 CA trust chain의 변경 기록을 중앙 계정에 보존한다.
- policy를 production에 적용하기 전에 별도의 account와 OU에서 IaC 및 application compatibility를 시험한다.
- key를 차단한 뒤에도 조사, disable과 deletion에 필요한 관리 API는 사용할 수 있게 둔다.
- AWS service 밖에서 EC2, container 또는 Lambda 내부의 OpenSSL이 수행하는 서명은 SCP와 IAM만으로 통제할 수 없으므로 host 및 software supply chain control이 별도로 필요하다.

## 운영 관점

먼저 현재 account와 Region에 존재하는 KMS key와 Private CA를 inventory하고, 각 key의 owner, workload, data lifetime과 규제 요구를 확인해야 한다. AWS KMS는 key spec에 대한 server-side 목록 filter를 제공하지 않으므로 `ListKeys` 결과를 각각 `DescribeKey`로 확인하는 inventory 절차가 필요하다.

강한 root-level deny를 한 번에 적용하면 legitimate test와 기존 deployment pipeline이 실패할 수 있다. audit 결과를 기반으로 sandbox OU, production OU, organization root 순으로 범위를 넓히고 access denied event와 failed deployment를 관찰하는 방식이 안전하다.

운영 중에는 다음 항목을 정기적으로 점검한다.

- 승인된 algorithm과 parameter set이 최신 조직 표준과 일치하는가?
- 새 AWS KMS operation이 key 사용 경로에 추가되었는가?
- 새로운 parameter set이 출시되어 deny list 밖에서 허용되고 있지 않은가?
- permission boundary가 모든 대상 role에 자동 적용되는가?
- CloudTrail과 Config 탐지 결과가 실제 incident workflow로 연결되는가?
- 기존 비승인 key의 disable, rotation 또는 deletion 계획이 완료되었는가?

## 시험 관점 정리

- AWS KMS는 encryption key뿐 아니라 asymmetric signing key의 생성과 사용도 관리한다.
- ML-DSA는 NIST FIPS 204로 표준화된 post-quantum digital signature 방식이다.
- `kms:KeySpec`은 key material의 종류를 나타내며 ML-DSA parameter set을 구분할 수 있다.
- `kms:SigningAlgorithm`의 `ML_DSA_SHAKE_256` 값은 세 ML-DSA variant를 서로 구분하지 못한다.
- SCP는 permission을 부여하지 않고 AWS Organization 내 account의 최대 permission boundary를 설정한다.
- explicit deny는 applicable policy의 allow보다 우선한다.
- SCP는 organization management account에는 적용되지 않는다.
- IAM permission boundary는 identity-based policy가 부여할 수 있는 최대 권한을 제한한다.
- AWS CloudTrail은 API activity를 기록하고 AWS Config는 resource configuration과 compliance를 평가한다.
- preventive control은 잘못된 action을 사전에 차단하고 detective control은 발생한 이탈을 발견한다.
- cryptographic agility는 표준 변화에 따라 algorithm과 key를 교체할 수 있는 능력이다.
- defense in depth는 SCP, IAM, key policy, provisioning control과 monitoring을 여러 계층에 적용하는 방식이다.

## 같이 보면 좋은 학습 포인트

- NIST FIPS 204와 ML-DSA 기본 구조
- AWS KMS asymmetric key와 digital signature
- `kms:KeySpec` IAM condition key
- AWS Organizations SCP 평가 방식
- IAM permission boundary와 identity policy 차이
- AWS Private CA와 certificate trust chain
- CloudTrail organization trail 중앙화
- AWS Config custom rule 작성
- software supply chain의 artifact와 SBOM signing
- cryptographic inventory와 post-quantum migration plan
- default-deny allow list와 deny list의 차이
- multi-account security governance

## 한 줄 정리

AWS의 ML-DSA 도입은 알고리즘 선택에서 끝나는 것이 아니라 `kms:KeySpec` 기반 SCP와 IAM 통제, 승인된 Private CA provisioning, CloudTrail 및 Config 탐지를 함께 운영해야 조직 전체의 암호 정책과 cryptographic agility를 유지할 수 있다.
