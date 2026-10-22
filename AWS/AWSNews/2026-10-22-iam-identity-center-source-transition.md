# IAM Identity Center의 Active Directory identity source 전환 개선

- 정리 날짜: 2026-10-22
- 원문 게시일: 2026-10-20
- 공식 출처: [AWS What's New - AWS IAM Identity Center simplifies identity source changes from Active Directory](https://aws.amazon.com/about-aws/whats-new/2026/10/iam-identity-center-chg-id-src-from-ad/)

## 핵심 내용

AWS IAM Identity Center가 Active Directory를 사용하던 organization instance의 identity source를 외부 identity provider(IdP) 또는 Identity Center directory로 변경할 때 기존 workforce identity와 access assignment를 유지하도록 개선됐다.

보존되는 항목은 다음과 같다.

- user
- group
- group membership
- permission set assignment
- application assignment

기존 방식에서는 Active Directory에서 Okta, Microsoft Entra ID 같은 외부 IdP 또는 Identity Center directory로 전환할 때 이러한 정보를 다시 구성해야 했다. 이제 기존 identity store가 유지되므로 user ID, group ID와 identity store ID가 바뀌지 않는다.

그 결과 user는 기존 AWS account와 application access를 유지하고, user ID나 group ID를 직접 참조하는 resource policy도 같은 identifier를 계속 사용할 수 있다.

## 왜 중요한가

identity source 전환은 단순한 login 화면 변경이 아니다. 대규모 organization에서는 수많은 user와 group이 여러 AWS account의 permission set, AWS managed application과 customer managed application에 연결되어 있다.

전환 과정에서 assignment가 삭제되면 access restoration 동안 업무가 중단될 수 있고, 누락이나 과도한 권한이 생기면 보안 사고로 이어질 수 있다. 특히 principal ID를 직접 참조하는 policy는 identity가 새 ID로 만들어질 경우 정상적으로 동작하지 않을 수 있다.

이번 개선은 기존 identity store와 identifier를 유지해 migration의 가장 위험한 재생성 단계를 줄인다. 조직은 legacy Active Directory 의존성을 줄이거나 cloud IdP로 전환하면서도 AWS access model을 다시 만들 필요가 적어진다.

다만 assignment 보존이 전체 migration의 자동 완료를 뜻하지는 않는다. 새 IdP의 authentication, provisioning, attribute mapping과 user lifecycle이 올바르게 구성되어야 실제 사용자가 계속 로그인할 수 있다.

## 변경 전과 변경 후

### 기존 전환 방식

Active Directory에서 다른 identity source로 바꿀 때 다음 작업이 필요할 수 있었다.

1. user, group와 membership을 inventory한다.
2. account별 permission set assignment를 export한다.
3. application assignment를 기록한다.
4. 새 identity source에서 principal을 provision한다.
5. name 또는 attribute를 기준으로 기존 principal과 새 principal을 매핑한다.
6. account와 application assignment를 다시 생성한다.
7. 누락과 extra permission을 비교한다.

principal matching이 잘못되거나 provisioning이 늦어지면 user access가 끊기고 rollback도 복잡해진다.

### 개선된 전환 방식

Active Directory에서 외부 IdP 또는 Identity Center directory로 전환해도 Identity Center의 기존 identity store가 그대로 유지된다.

1. 기존 user와 group ID가 유지된다.
2. group membership이 유지된다.
3. permission set assignment가 유지된다.
4. application assignment가 유지된다.
5. 기존 ID를 참조하는 policy가 계속 같은 principal을 가리킨다.
6. 새 identity source의 표준 sign-in 설정을 완료하면 user가 기존 access를 이어서 사용할 수 있다.

## 서비스 및 아키텍처 관점

전환 후 access 흐름은 다음과 같이 이해할 수 있다.

1. workforce user가 새 외부 IdP 또는 Identity Center directory에서 인증한다.
2. 외부 IdP를 사용하는 경우 SAML 2.0 federation이 authentication assertion을 IAM Identity Center에 전달한다.
3. SCIM 또는 지원되는 provisioning 방식이 user와 group attribute 및 lifecycle을 동기화한다.
4. IAM Identity Center는 유지된 identity store에서 기존 user와 group ID를 사용한다.
5. 기존 permission set assignment가 대상 AWS account의 IAM role access를 제공한다.
6. 기존 application assignment가 AWS managed application 또는 customer managed application access를 제공한다.
7. user는 dual-stack AWS access portal URL을 통해 account와 application을 선택한다.

여기서 authentication과 authorization을 구분해야 한다. 새 IdP는 사용자의 신원을 인증하지만, 어떤 AWS account와 application에 접근할 수 있는지는 IAM Identity Center에 유지된 assignment가 결정한다.

이 기능은 **Active Directory에서 외부 IdP 또는 Identity Center directory로 변경하는 방향**에 초점을 둔다. identity source를 Active Directory 방향으로 변경하는 동작은 기존 방식과 동일하므로 양방향 전환이 모두 같은 방식으로 보존된다고 가정하면 안 된다.

## 보안 관점

- migration 전에 break-glass administrator access를 별도로 확인해 SSO 문제 시에도 복구할 수 있게 한다.
- 새 IdP의 SAML signing certificate, issuer, audience와 ACS URL을 정확하게 검증한다.
- SCIM token은 secret으로 관리하고 최소 권한, rotation과 만료 절차를 적용한다.
- user name, email, immutable identifier와 group naming rule이 기존 identity와 충돌하지 않는지 확인한다.
- MFA와 conditional access policy가 새 sign-in path에서도 조직 기준을 만족하는지 시험한다.
- 기존 ID와 assignment가 유지되므로 오래된 group membership이나 불필요한 access도 함께 유지될 수 있다. 전환 전에 access review를 수행한다.
- CloudTrail과 IdP audit log를 함께 수집해 sign-in, provisioning과 assignment 변경을 추적한다.
- 전환 직후 administrator, developer, read-only와 application user 등 대표 역할별 end-to-end access를 검증한다.
- resource policy가 user ID, group ID 또는 identity store ID를 직접 참조한다면 실제 identifier가 유지되었는지 확인한다.
- 외부 IdP 장애 시 authentication이 중단될 수 있으므로 federation dependency와 emergency access 절차를 문서화한다.

## 운영 관점

assignment 재생성 작업은 줄었지만 production identity migration에는 여전히 change window와 검증 계획이 필요하다.

권장 절차는 다음과 같다.

1. 현재 user, group, membership, permission set과 application assignment를 export한다.
2. 사용하지 않는 principal과 과도한 permission을 정리한다.
3. 새 identity source에서 attribute와 group mapping을 구성한다.
4. test environment 또는 제한된 user group으로 authentication과 provisioning을 검증한다.
5. break-glass access와 rollback decision point를 확인한다.
6. 낮은 사용 시간에 identity source를 변경한다.
7. dual-stack AWS access portal URL과 새 login 방법을 사용자에게 안내한다.
8. 대표 user가 각 AWS account role과 application에 접근할 수 있는지 확인한다.
9. 전환 전 inventory와 현재 상태를 비교해 누락 및 의도하지 않은 access를 찾는다.
10. 일정 기간 동안 sign-in failure, SCIM error와 help desk incident를 집중 monitoring한다.

기존 user와 assignment가 유지된다는 이유로 backup을 생략하면 안 된다. migration 전 inventory는 보존 기능이 예상대로 동작했는지 확인하는 기준이자 access review 자료가 된다.

IAM Identity Center는 이 기능을 organization instance에서 지원한다. 모든 IAM Identity Center 지원 AWS Region에서 사용할 수 있으며 IAM Identity Center 자체에는 추가 요금이 없다.

## 시험 관점 정리

- AWS IAM Identity Center는 workforce의 AWS account와 application access를 중앙 관리한다.
- identity source는 user와 group identity의 authoritative source다.
- Identity Center directory, Active Directory와 external IdP를 identity source로 사용할 수 있다.
- SAML 2.0은 외부 IdP와 service provider 사이의 authentication federation에 사용된다.
- SCIM은 user와 group의 provisioning 및 lifecycle synchronization에 사용된다.
- permission set은 하나 이상의 IAM policy를 묶어 AWS account access 수준을 정의한다.
- permission set assignment는 user 또는 group을 특정 account의 permission set과 연결한다.
- AWS IAM Identity Center는 assignment를 기반으로 target account에 `AWSReservedSSO_` IAM role을 provision한다.
- authentication은 사용자가 누구인지 확인하고 authorization은 무엇을 할 수 있는지 결정한다.
- stable identifier를 유지하면 principal ID를 참조하는 policy와 application mapping의 변경을 줄일 수 있다.
- organization instance는 AWS Organizations와 연결해 여러 account의 workforce access를 관리한다.
- break-glass access는 federation 장애 시 사용할 수 있는 독립적인 emergency administrator 경로다.

## 같이 보면 좋은 학습 포인트

- IAM Identity Center organization instance
- Identity Center directory와 AWS Managed Microsoft AD
- SAML 2.0 federation flow
- SCIM user 및 group provisioning
- permission set과 account assignment
- AWS Organizations multi-account access
- external IdP의 MFA와 conditional access
- CloudTrail identity event auditing
- least privilege와 periodic access review
- break-glass administrator account
- identity migration runbook과 rollback plan
- AWS access portal과 dual-stack endpoint

## 한 줄 정리

IAM Identity Center는 Active Directory에서 외부 IdP나 Identity Center directory로 전환할 때 기존 user ID, group, membership과 account 및 application assignment를 유지해 workforce identity migration의 중단 위험과 재구성 작업을 크게 줄였다.
