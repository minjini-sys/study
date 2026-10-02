# AWS Well-Architected Agent 프리뷰 공개

- 정리 날짜: 2026-10-02
- 원문 게시일: 2026-10-01
- 공식 출처: [AWS News Blog - Announcing AWS Well-Architected Agent, an AI-powered intelligence to optimize your cloud environment (preview)](https://aws.amazon.com/blogs/aws/announcing-aws-well-architected-agent-an-ai-powered-intelligence-to-optimize-your-cloud-environment-preview/)
- 추가 출처: [AWS What's New - AWS Well-Architected Agent is now available in preview](https://aws.amazon.com/about-aws/whats-new/2026/10/aws-well-architected-agent/)

## 핵심 내용

AWS가 클라우드 환경을 분석하고 비용, 보안, 성능과 복원력 개선 방안을 제안하는 **AWS Well-Architected Agent**를 공개 프리뷰로 출시했다. 이 서비스는 AWS Trusted Advisor와 AWS Well-Architected Tool의 다음 세대 발전 형태로, 일반적인 체크리스트 대신 실제 리소스 구성, 사용량 지표, 애플리케이션 토폴로지와 사용자가 선언한 비즈니스 목표를 함께 분석한다.

Well-Architected Agent는 65개가 넘는 AWS 서비스의 구성을 Well-Architected 모범 사례와 비교하고, 권장 사항을 영향도와 구현 노력에 따라 우선순위화한다. 각 결과에는 가능한 경우 콘솔 단계, AWS CLI 명령, Systems Manager Automation runbook 또는 수정된 Infrastructure as Code 코드가 함께 제공된다.

Terraform, AWS CloudFormation과 AWS CDK 프로젝트를 업로드해 배포 전에 아키텍처를 검토할 수도 있다. 현재 권장 사항은 리소스, 애플리케이션, 전체 아키텍처라는 세 단계로 제공된다.

## 왜 중요한가

기존의 수동 Well-Architected Review는 담당자가 질문에 답하고 여러 서비스의 설정과 지표를 직접 연결해야 했다. 규모가 커질수록 모든 워크로드를 정기적으로 검토하고 개선 사항의 우선순위를 정하기 어렵다.

Well-Architected Agent는 운영 환경과 IaC를 지속적으로 분석해 실제 비즈니스 목표에 맞는 개선 순서를 제시한다. 또한 한 영역을 개선할 때 다른 영역에 생기는 영향을 보여준다. 예를 들어 데이터베이스에 Multi-AZ 장애 조치를 추가하면 복원력은 높아지지만 비용과 성능에도 영향을 줄 수 있는데, 이런 교차 영역의 trade-off를 변경 전에 확인할 수 있다.

## 주요 기능

### 목표 기반 권장 사항

비용 최적화, 성능, 복원력과 보안 중 조직이 중요하게 생각하는 목표를 설정한다. Agent는 모든 결과를 동일하게 나열하지 않고 목표에 대한 영향과 적용 노력에 따라 순서를 정한다.

### 세 단계 분석

- **리소스 수준**: 개별 리소스의 설정 문제, 예상 비용 영향과 구체적인 수정 단계를 제공한다.
- **애플리케이션 수준**: 여러 리소스의 관계를 하나의 애플리케이션 범위에서 종합한다.
- **아키텍처 수준**: 전체 설계 패턴과 Well-Architected 원칙의 차이를 분석하고 필요한 IaC 변경을 제안한다.

### 실행 가능한 수정안

권장 사항에 따라 안내형 콘솔 절차, CLI 명령, SSM Automation runbook 또는 Terraform, CloudFormation, CDK 변경 코드를 선택할 수 있다. 콘솔과 API를 모두 지원하므로 기존 개발 및 운영 워크플로에 결과를 연결할 수 있다.

## 서비스 및 아키텍처 관점

1. 사용자가 Agent profile에서 분석할 AWS 계정, 리전, 애플리케이션과 최적화 영역을 정한다.
2. 고객 관리형 IAM 역할을 통해 Agent가 리소스 구성, 사용량 지표와 애플리케이션 토폴로지를 읽는다.
3. 태그, 계정, 리전과 서비스 정보를 이용해 리소스를 애플리케이션 단위로 묶고 업무 맥락을 추가한다.
4. Well-Architected 모범 사례와 목표를 기준으로 결과를 분석하고 우선순위를 계산한다.
5. 리소스 수정 절차와 함께 다른 영역에 미치는 영향 및 위험을 제시한다.
6. 운영자는 제안 내용을 검토한 뒤 콘솔, CLI, SSM Automation 또는 IaC 변경으로 적용하고 결과를 검증한다.

IaC 검토는 운영 리소스를 변경하기 전에 문제를 찾는 shift-left 방식이다. 운영 환경 분석과 배포 전 코드 검토를 함께 사용하면 설계, 배포와 지속적인 운영 개선을 하나의 흐름으로 연결할 수 있다.

## 보안 및 운영 관점

- Agent profile용 IAM 역할에는 분석에 필요한 읽기 권한만 부여하고 수정 권한과 분리한다.
- 분석 범위를 계정, 리전, 서비스와 태그로 제한해 불필요한 리소스 접근을 줄인다.
- 제안된 CLI, runbook과 IaC 변경은 프로덕션에 바로 적용하지 않고 코드 리뷰와 비운영 환경 검증을 거친다.
- 생성형 AI 기반 권장 사항에는 오류나 누락이 있을 수 있으므로 최종 결정과 변경 책임은 운영자가 가진다.
- 비용, 보안, 성능과 복원력 사이의 trade-off를 검토하고 조직의 위험 허용 수준에 맞춰 적용한다.
- 권장 사항을 적용한 뒤 CloudWatch 지표, AWS Config와 CloudTrail을 통해 효과 및 변경 이력을 확인한다.
- 기존 AWS Well-Architected Tool은 사용자 정의 lens를 이용한 수동 검토가 필요할 때 계속 사용할 수 있다.
- 프리뷰 서비스이므로 지원 리전, 기능 제한과 변경 가능성을 프로덕션 도입 전에 확인한다.

## 지원 범위 및 주의점

- Well-Architected Agent 자체와 권장 사항은 미국 동부 버지니아 북부, 미국 동부 오하이오, 미국 서부 오리건 리전에서 제공된다.
- 모든 AWS 상용 리전의 워크로드를 분석 대상으로 등록할 수 있다.
- AWS Support를 통해 제공되며 AWS Support 플랜이 있는 고객이 사용할 수 있다.
- Agent profile을 만든 뒤 리소스 및 애플리케이션 권장 사항이 생성되기까지 최대 24시간이 걸릴 수 있다.
- 공개 프리뷰이므로 정식 출시 전에 기능, 제공 범위와 사용 조건이 바뀔 수 있다.

## 시험 관점 정리

- AWS Well-Architected Framework의 핵심 영역에는 운영 우수성, 보안, 안정성, 성능 효율성, 비용 최적화와 지속 가능성이 있다.
- AWS Well-Architected Tool은 워크로드를 lens와 질문에 따라 검토하고 개선 항목을 기록한다.
- AWS Trusted Advisor는 계정의 비용, 성능, 보안, 내결함성과 서비스 한도 관련 점검을 제공한다.
- Systems Manager Automation은 runbook을 사용해 반복적인 운영 및 수정 작업을 자동화한다.
- IaC를 사용하면 인프라 변경을 버전 관리하고 반복 가능하게 배포할 수 있다.
- Multi-AZ는 가용성과 복원력을 높이지만 추가 비용 등 다른 영역과의 trade-off를 함께 고려해야 한다.
- 최소 권한 IAM 역할, 변경 검토와 사후 검증은 AI가 수정안을 제안하더라도 그대로 유지해야 하는 운영 원칙이다.

## 같이 보면 좋은 학습 포인트

- AWS Well-Architected Framework의 6개 영역
- AWS Well-Architected Tool과 사용자 정의 lens
- AWS Trusted Advisor의 검사 범주와 Support 플랜
- AWS Systems Manager Automation runbook
- Terraform, CloudFormation과 AWS CDK 비교
- IAM 역할의 신뢰 정책과 최소 권한 설계
- AWS Resource Groups 및 태그 기반 애플리케이션 분류
- AWS Config, CloudTrail과 CloudWatch를 이용한 변경 검증
- 생성형 AI 권장 사항에 대한 human-in-the-loop 운영

## 한 줄 정리

AWS Well-Architected Agent는 실제 리소스와 IaC를 비즈니스 목표 및 Well-Architected 모범 사례에 맞춰 분석하고, 우선순위가 지정된 실행 가능한 개선안을 제공하는 AI 기반 아키텍처 최적화 서비스다.
