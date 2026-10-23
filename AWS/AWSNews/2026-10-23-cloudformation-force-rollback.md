# AWS CloudFormation Force Rollback으로 실패한 스택 복구하기

- 정리 날짜: 2026-10-23
- 원문 게시일: 2026-10-22
- 공식 출처: [AWS What's New - AWS CloudFormation now simplifies stack update rollback operations](https://aws.amazon.com/about-aws/whats-new/2026/10/aws-cloudformation-cdk/)
- 공식 문서: [AWS CloudFormation User Guide - Continue rolling back an update](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-continueupdaterollback.html)

## 핵심 내용

AWS CloudFormation이 stack update rollback을 한 번의 operation으로 완료할 수 있는 **force rollback** 기능을 출시했다.

CloudFormation stack update가 실패하면 서비스는 기존의 정상 상태로 resource를 되돌린다. 하지만 resource가 CloudFormation 외부에서 삭제되거나 수동으로 변경된 경우 rollback도 실패해 stack이 `UPDATE_ROLLBACK_FAILED` 상태에 멈출 수 있다. 이 상태의 stack에는 새로운 update를 배포할 수 없다.

기존에는 운영자가 rollback에 실패한 resource의 logical ID를 찾아 `--resources-to-skip`에 직접 나열해야 했다. nested stack이 여러 단계라면 child stack 이름과 resource logical ID까지 조사해야 했다.

이제 `continue-update-rollback` operation에 `--force-rollback`을 사용하면 CloudFormation이 rollback에 실패한 resource를 자동으로 건너뛰고 stack을 `UPDATE_ROLLBACK_COMPLETE` 상태로 되돌린다. AWS CLI, SDK, AWS CDK와 AI agent를 포함한 CloudFormation workflow에서 사용할 수 있다.

## 왜 중요한가

Infrastructure as Code 배포 실패 자체보다 더 위험한 상황은 rollback도 실패해 stack이 잠기는 것이다. production incident 중에는 service 복구와 다음 배포가 모두 막히며, nested stack의 logical ID를 수동으로 조사하는 동안 recovery time이 길어진다.

force rollback은 stack을 빠르게 다시 배포 가능한 상태로 전환해 운영자가 application 복구와 후속 조치에 집중하도록 한다. 특히 resource 수가 많거나 nested stack hierarchy가 깊은 환경에서 수동 조사 시간을 줄일 수 있다.

그러나 이 기능은 실패 원인을 해결하거나 실제 resource를 template 상태로 되돌리는 기능이 아니다. 실패한 resource를 skip하여 CloudFormation workflow를 계속 진행하는 recovery mechanism이다. 건너뛴 resource는 실제 상태와 template이 달라질 수 있으므로 후속 drift 확인과 reconciliation이 필수다.

## 기존 방식과 Force Rollback 비교

### 기존 `resources-to-skip` 방식

운영자가 다음 작업을 직접 수행한다.

1. stack event에서 rollback에 실패한 resource를 찾는다.
2. 해당 resource가 forward update가 아니라 rollback 중 `UPDATE_FAILED`가 되었는지 확인한다.
3. root stack resource는 logical ID를 기록한다.
4. nested stack resource는 `NestedStackName.ResourceLogicalID` 형식으로 기록한다.
5. 필요한 최소 resource만 `--resources-to-skip`에 지정한다.
6. rollback을 다시 진행한다.

특정 resource만 명시적으로 건너뛰고 싶을 때는 여전히 이 방식이 유용하다.

### 새로운 `force-rollback` 방식

다음 명령으로 CloudFormation이 rollback 실패 resource를 자동으로 식별하고 건너뛴다.

```bash
aws cloudformation continue-update-rollback \
  --stack-name MyStack \
  --force-rollback
```

`--force-rollback`과 `--resources-to-skip`은 같은 request에서 함께 사용할 수 없다. 전체 hierarchy에서 rollback 실패 resource를 자동으로 건너뛰려면 force rollback을 사용하고, 대상 resource를 직접 통제하려면 `--resources-to-skip`을 사용한다.

## 동작 방식

1. stack이 update 실패 후 rollback을 시작한다.
2. 하나 이상의 resource가 이전 상태로 돌아가지 못한다.
3. stack status가 `UPDATE_ROLLBACK_FAILED`가 된다.
4. 운영자가 원인을 즉시 해결할 수 없으면 force rollback을 실행한다.
5. CloudFormation이 rollback 중 `UPDATE_FAILED` 상태가 된 resource를 자동으로 건너뛴다.
6. nested stack 안에서 실패한 resource도 같은 방식으로 처리한다.
7. 건너뛴 resource는 `UPDATE_COMPLETE`로 표시되고 status reason에 `Resource skipped during UpdateRollback`이 기록된다.
8. rollback이 끝나면 stack이 `UPDATE_ROLLBACK_COMPLETE` 상태가 되어 다시 update할 수 있다.
9. 운영자는 건너뛴 resource와 template의 차이를 확인하고 다음 배포 전에 일치시킨다.

forward update 단계에서 실패한 resource는 force rollback의 skip 대상이 아니다. rollback 과정 자체에서 되돌리지 못한 resource만 자동으로 건너뛴다.

## 서비스 및 아키텍처 관점

CloudFormation은 template을 desired state로 사용해 resource lifecycle을 관리한다. 정상적인 update 흐름은 change set 검토, update 실행, 성공 또는 rollback 순서로 진행된다.

force rollback 이후에는 stack status만 deployable state로 돌아왔을 뿐 다음 세 상태가 서로 다를 수 있다.

- CloudFormation template의 desired state
- CloudFormation이 기록한 resource state
- AWS account에 실제로 존재하는 resource state

따라서 recovery workflow는 force rollback에서 끝나면 안 된다.

1. stack event와 skipped resource를 기록한다.
2. 개별 resource drift detection을 실행한다.
3. 수동 변경과 out-of-band deletion 여부를 조사한다.
4. 실제 resource를 template에 맞추거나 template을 승인된 실제 상태에 맞게 수정한다.
5. drift-aware change set을 생성해 다음 update의 영향을 검토한다.
6. change set을 적용해 CloudFormation과 실제 resource를 다시 일치시킨다.

nested stack에서는 parent rollback이 child stack도 되돌리려고 시도한다. force rollback을 사용하면 여러 nested level의 실패 resource를 직접 나열하지 않아도 전체 hierarchy의 rollback을 완료할 수 있다.

## 보안 관점

- force rollback permission은 production stack을 운영하는 제한된 deployment role과 break-glass role에만 부여한다.
- CloudTrail에서 `ContinueUpdateRollback` 호출자, 시간, stack과 option을 감사한다.
- skip된 security group, IAM policy, KMS key policy와 network resource는 보안 기준에서 벗어날 수 있으므로 우선 검토한다.
- drift 상태의 resource를 장기간 방치하지 않고 incident ticket과 owner를 지정한다.
- CI/CD pipeline에서 force rollback을 무조건 자동 실행하지 않는다. 사람의 승인과 실패 원인 확인 단계를 둔다.
- change set을 통해 다음 배포가 권한, network exposure와 data resource에 미칠 영향을 확인한다.
- stack policy와 IAM least privilege를 사용해 중요 resource에 대한 의도하지 않은 update를 제한한다.
- CloudFormation 외부의 수동 변경을 줄이고 필요한 emergency change는 기록과 reconciliation 절차를 갖춘다.

## 운영 관점

force rollback은 incident recovery 시간을 줄이지만 정상적인 배포 성공으로 취급해서는 안 된다. monitoring과 deployment pipeline은 `UPDATE_ROLLBACK_COMPLETE`만 보는 것이 아니라 skipped resource의 존재와 stack status reason도 확인해야 한다.

운영 runbook에는 다음 항목을 포함하는 것이 좋다.

- 최초 update 실패 원인과 rollback 실패 원인을 구분한다.
- resource를 정상적으로 복구할 수 있으면 force rollback 전에 원인을 해결한다.
- 즉시 수정할 수 없고 stack unlock이 우선인 경우에만 force rollback을 사용한다.
- rollback 완료 후 skipped resource 목록과 owner를 기록한다.
- `DetectStackResourceDrift`로 개별 resource 상태를 확인한다.
- drift-aware change set을 검토하기 전에는 다음 production deployment를 진행하지 않는다.
- data resource는 rollback 또는 재생성 시 데이터 손실 가능성을 별도로 평가한다.
- nested stack과 dependency가 정상 상태인지 parent와 child level에서 모두 확인한다.

공식 문서에 따르면 일반 drift detection에서는 skipped resource가 `NOT_CHECKED`로 보고될 수 있다. 실제 차이를 확인하려면 해당 resource에 `DetectStackResourceDrift`를 실행해야 한다.

force rollback은 모든 AWS Region에서 추가 요금 없이 제공된다.

## 주의해야 할 상황

- 삭제된 database처럼 CloudFormation이 이전 resource가 존재한다고 가정하면 rollback이 실패할 수 있다.
- resource에 대한 수동 변경으로 이전 property를 다시 적용할 수 없을 수 있다.
- service quota, permission 부족과 dependency 문제로 rollback operation이 실패할 수 있다.
- skip된 resource는 template과 불일치하므로 이후 update가 다시 실패할 수 있다.
- 불일치를 해결하지 않고 반복 update하면 stack이 복구하기 어려운 상태가 될 수 있다.
- dependent resource가 함께 실패했어도 root cause resource만 해결하면 나머지는 자동 복구될 수 있으므로 수동 skip 방식에서는 최소 resource만 지정한다.

## 시험 관점 정리

- AWS CloudFormation은 template 기반으로 AWS resource를 생성하고 관리하는 Infrastructure as Code service다.
- stack은 하나의 unit으로 관리되는 AWS resource collection이다.
- change set은 stack update 전에 예상 변경 사항을 보여준다.
- update 실패 시 CloudFormation은 이전의 known good state로 rollback을 시도한다.
- `UPDATE_ROLLBACK_FAILED` 상태의 stack은 새로운 update를 받을 수 없다.
- `continue-update-rollback`은 실패한 update rollback을 다시 진행한다.
- `--force-rollback`은 rollback에 실패한 resource를 자동으로 skip한다.
- `--resources-to-skip`은 운영자가 건너뛸 logical ID를 직접 지정한다.
- force rollback과 resources-to-skip은 동시에 사용할 수 없다.
- nested stack은 parent template에서 `AWS::CloudFormation::Stack` resource로 정의된다.
- drift는 template의 expected configuration과 실제 resource configuration의 차이다.
- rollback 완료와 resource consistency 복구는 서로 다른 단계다.

## 같이 보면 좋은 학습 포인트

- CloudFormation stack lifecycle과 status
- change set 기반 배포
- stack event troubleshooting
- `UPDATE_FAILED`와 `UPDATE_ROLLBACK_FAILED` 차이
- nested stack logical ID 구조
- CloudFormation drift detection
- drift-aware change set
- stack policy와 termination protection
- CloudTrail을 이용한 IaC 감사
- CI/CD deployment approval gate
- break-glass 운영 절차
- Infrastructure as Code 외부 변경 통제

## 한 줄 정리

CloudFormation force rollback은 rollback 실패 resource를 자동으로 건너뛰어 `UPDATE_ROLLBACK_FAILED` stack을 빠르게 다시 배포 가능한 상태로 만들지만, skip된 resource와 template의 drift를 다음 배포 전에 반드시 확인하고 해소해야 한다.
