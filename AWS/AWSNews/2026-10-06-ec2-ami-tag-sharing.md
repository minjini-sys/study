# EC2 AMI Tag Sharing으로 계정 간 태그 공유 지원

- 정리 날짜: 2026-10-06
- 원문 게시일: 2026-10-05
- 공식 출처: [AWS Compute Blog - Introducing EC2 AMI Tag Sharing: Share EC2 tags across AWS accounts](https://aws.amazon.com/blogs/compute/introducing-ec2-ami-tag-sharing-share-ec2-tags-across-aws-accounts/)

## 핵심 내용

Amazon EC2가 공유 AMI의 태그를 다른 AWS 계정에도 전달하는 **EC2 AMI Tag Sharing** 기능을 공개했다. AMI 소유자가 태그 키에 `ec2:SharedTag/` 접두사를 붙이면 해당 태그가 AMI에 접근할 수 있는 모든 계정에 자동으로 표시된다.

AMI를 개별 계정, AWS Organizations 범위 또는 공개 이미지로 공유해도 같은 방식으로 동작한다. 소유 계정에서 공유 태그 값을 수정하면 수신 계정에도 자동으로 반영되므로 SNS와 Lambda를 이용한 별도 태그 복제 파이프라인이 필요하지 않다.

접두사가 없는 기존 태그는 계속 소유 계정에만 보인다. 공유 태그는 소유자만 생성, 수정 또는 삭제할 수 있고 AMI를 받은 계정은 값을 읽을 수 있지만 변경할 수 없다.

## 왜 중요한가

여러 계정을 사용하는 조직은 중앙 image factory에서 보안 강화와 패치를 마친 golden AMI를 만든 뒤 수백 개의 워크로드 계정에 공유한다. 하지만 기존에는 AMI만 공유되고 승인 상태, 운영체제 버전, 패치 날짜 같은 태그는 전달되지 않았다.

이 문제를 해결하기 위해 각 계정에 SNS와 Lambda 기반 복제 로직을 구축하면 AMI와 리전, 계정 수만큼 API 호출이 증가한다. 호출 제한, Lambda 실패와 tag drift가 발생하면 계정마다 같은 AMI를 서로 다른 상태로 판단할 수도 있다.

AMI Tag Sharing은 메타데이터의 원본을 소유 계정 하나로 유지하고 변경을 자동 전파한다. FinOps 분류, 보안 승인, 인벤토리와 ABAC 정책에 필요한 정보를 중앙에서 일관되게 관리할 수 있다.

## 동작 방식

- `ec2:SharedTag/status=approved`처럼 접두사가 있는 태그는 AMI 수신 계정에 보인다.
- `team=platform-engineering`처럼 일반 태그는 소유 계정에만 남는다.
- 수신 계정은 공유 태그를 변경할 수 없지만 자체 private tag는 추가할 수 있다.
- 공유 태그와 private tag는 모두 소유자의 리소스당 50개 태그 한도에 포함된다.
- 공유 태그는 수신 계정의 태그 한도를 사용하지 않는다.
- AMI 복사 시 `--copy-image-tags`를 지정하면 공유 태그가 새 이미지로 복사된다.
- 출시 시점에는 AMI 리소스만 Tag Sharing을 지원한다.

예를 들어 다음 태그는 AMI의 승인 상태를 모든 수신 계정에 전달한다.

```bash
aws ec2 create-tags \
  --resources ami-0abcdef1234567890 \
  --tags Key=ec2:SharedTag/status,Value=approved
```

공유를 중단하려면 접두사가 붙은 태그를 삭제한다. 같은 정보를 소유 계정에서만 유지하려면 접두사 없는 private tag로 다시 생성한다.

## 서비스 및 아키텍처 관점

1. 중앙 image factory 계정이 EC2 Image Builder 또는 자체 파이프라인으로 golden AMI를 생성한다.
2. 보안 검사와 패치 검증이 끝나면 `ec2:SharedTag/status=approved`와 패치 날짜 등의 공유 태그를 추가한다.
3. AMI를 조직 또는 지정된 워크로드 계정에 공유한다.
4. 수신 계정은 공유 태그를 조회해 승인된 이미지인지 판단하고 인스턴스를 시작한다.
5. 이미지가 오래되거나 취약해지면 소유 계정이 상태 태그를 변경하고 모든 수신 계정에 새 값이 전파된다.
6. 수신 계정의 IAM, ABAC 정책과 자동화는 공유 태그를 기준으로 실행 허용 여부를 결정할 수 있다.

이 구조에서는 AMI 소유 계정이 이미지와 공유 메타데이터의 신뢰 원천이 된다. 수신 계정은 복제 파이프라인을 운영하는 대신 신뢰할 수 있는 provider 계정과 태그 계약을 검증해야 한다.

## 보안 및 운영 관점

- 공유 태그는 AMI에 접근 가능한 모든 계정에 노출되므로 개인정보, 비밀 값과 내부 기밀을 넣지 않는다.
- 수신 계정이 공유 태그 기반 ABAC를 사용한다면 정책의 태그 키도 `ec2:SharedTag/` 접두사를 포함하도록 검증한다.
- 태그 기반 인스턴스 실행 정책은 소유 계정의 태그 변경에 영향을 받으므로 AMI 제공 계정을 신뢰 경계로 관리한다.
- Allowed AMIs에 신뢰할 수 있는 provider 계정 ID를 지정해 승인되지 않은 이미지의 검색 및 실행을 제한한다.
- Allowed AMIs는 강제 적용 전에 audit mode로 영향 범위를 확인한다.
- 공유 태그를 생성하거나 수정할 권한은 image pipeline 역할과 제한된 운영자에게만 부여한다.
- `status`, `patch-date`, `os-version` 같은 태그의 의미, 허용 값과 변경 절차를 조직 표준으로 문서화한다.
- CloudTrail에서 AMI 공유 권한과 태그 변경 이벤트를 기록하고 승인 상태 변경을 알림에 연결한다.
- 50개 태그 한도에 공유 태그도 포함되므로 불필요한 키와 중복 태그를 정리한다.
- AMI 복사 시 태그가 필요한지 검토하고 `--copy-image-tags` 사용 여부를 자동화 코드에 명시한다.

## 시험 관점 정리

- AMI는 EC2 인스턴스 시작에 필요한 운영체제와 소프트웨어 구성을 담은 이미지다.
- AMI는 특정 계정, 조직 또는 공개 범위로 공유할 수 있다.
- AWS 태그는 비용 할당, 자동화, 인벤토리와 ABAC 조건에 사용할 수 있다.
- ABAC는 사용자 또는 리소스의 속성을 정책 조건으로 사용해 접근을 제어한다.
- EC2 AMI Tag Sharing은 `ec2:SharedTag/` 접두사를 사용하며 일반 태그는 기존처럼 private 상태를 유지한다.
- 공유 태그의 쓰기 권한은 AMI 소유자에게 있고 수신자는 읽기만 가능하다.
- golden AMI 패턴은 중앙 계정에서 검증된 이미지를 생성하고 여러 워크로드 계정에 배포하는 방식이다.
- Allowed AMIs는 계정에서 사용할 수 있는 이미지의 기준을 설정하는 거버넌스 기능이다.

## 같이 보면 좋은 학습 포인트

- EC2 AMI 공유 권한과 AWS Organizations
- EC2 Image Builder 기반 golden AMI pipeline
- IAM 정책의 `aws:ResourceTag`와 요청 태그 조건
- ABAC와 RBAC의 차이
- Allowed AMIs의 audit mode 및 enabled mode
- AWS Resource Access Manager와 AMI 직접 공유의 차이
- CloudTrail을 이용한 `CreateTags`, `DeleteTags` 감사
- 다중 계정 landing zone의 image factory 계정 설계
- AMI 수명 주기, 취약점 검사와 폐기 자동화

## 한 줄 정리

EC2 AMI Tag Sharing은 `ec2:SharedTag/` 접두사로 승인 상태와 패치 정보 같은 AMI 메타데이터를 여러 계정에 자동 전파해 golden AMI 거버넌스와 ABAC 운영을 단순화한다.
