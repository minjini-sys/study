# AWS WAF, CloudFront 자동 JavaScript SDK 통합 지원

## 공식 출처

- [AWS WAF announces automatic SDK integration for Amazon CloudFront distributions](https://aws.amazon.com/about-aws/whats-new/2026/10/aws-waf-automatic-sdk-integration-amazon-cloudfront/)
- 발표일: 2026년 10월 30일

## 핵심 내용

AWS WAF가 Amazon CloudFront 배포를 위한 자동 JavaScript SDK 통합을 지원한다.

이 기능을 활성화하면 CloudFront가 제공하는 HTML에 AWS WAF JavaScript SDK의 스크립트 태그를 자동으로 추가한다. 애플리케이션 코드, 빌드 과정, 배포 절차를 직접 변경하지 않아도 클라이언트 측 신호를 AWS WAF에 제공할 수 있다.

자동 SDK 통합의 클라이언트 측 신호를 사용하는 AWS WAF 관리형 규칙 그룹은 다음과 같다.

- Bot Control의 Targeted inspection level
- Fraud Control account takeover prevention(ATP)
- Fraud Control account creation fraud prevention(ACFP)

기존에는 보호할 애플리케이션의 모든 페이지에 스크립트 태그를 직접 추가하고 릴리스마다 유지해야 했다. 이번 업데이트로 CloudFront 배포를 보호하는 Web ACL에서 통합을 활성화해 이 작업을 중앙에서 관리할 수 있다.

## 왜 중요한가

Bot Control과 Fraud Control은 서버로 들어온 HTTP 요청뿐 아니라 브라우저에서 수집한 클라이언트 측 신호를 활용해 자동화된 요청과 사기 행동을 더 정교하게 구분할 수 있다.

그러나 관리형 규칙 그룹을 Web ACL에 추가했더라도 애플리케이션 페이지에 SDK가 빠져 있으면 필요한 신호를 제공하지 못한다. 애플리케이션이 많거나 직접 수정하기 어려운 HTML을 제공하는 환경에서는 이런 구성 누락이 반복될 수 있다.

자동 SDK 통합은 다음과 같은 이점을 제공한다.

- 애플리케이션별 스크립트 삽입 작업 감소
- 코드와 배포 파이프라인을 변경하지 않는 중앙 관리
- 여러 애플리케이션에서 일관된 클라이언트 신호 수집
- 관리형 규칙 그룹과 SDK 구성 사이의 누락 위험 감소
- 신규 애플리케이션과 페이지에 대한 보호 적용 단순화

## 서비스 및 아키텍처 관점

이 기능은 다음 조건을 충족하는 Web ACL에 사용할 수 있다.

- Web ACL이 Amazon CloudFront에서 사용되고 있어야 한다.
- Web ACL에 Bot Control Targeted, ATP 또는 ACFP 관리형 규칙 그룹 중 하나 이상이 포함되어야 한다.

기본적인 요청 흐름은 다음과 같다.

1. 사용자가 CloudFront를 통해 웹 페이지를 요청한다.
2. CloudFront가 원본에서 받은 HTML 응답을 제공한다.
3. 자동 통합이 AWS WAF JavaScript SDK 스크립트 태그를 HTML에 추가한다.
4. 브라우저에서 SDK가 실행되어 클라이언트 측 신호를 생성한다.
5. AWS WAF 관리형 규칙 그룹이 요청 정보와 클라이언트 신호를 이용해 트래픽을 평가한다.

설정 방법은 AWS Management Console, AWS CLI, AWS SDK, AWS CDK, AWS CloudFormation을 지원한다. 다만 생성 방식에 따라 기본 동작이 다르다.

- AWS WAF 콘솔에서 대상 관리형 규칙 그룹을 포함한 신규 Web ACL을 생성하면 자동 SDK 통합이 기본 활성화된다.
- CLI, SDK, CDK 또는 CloudFormation 등 프로그래밍 방식으로 신규 Web ACL을 생성하면 기본 비활성화된다.
- 이번 기능 출시 전에 생성된 기존 Web ACL은 자동으로 변경되지 않는다.

따라서 콘솔과 인프라 코드로 만든 환경의 설정이 서로 다를 수 있다. 운영 표준을 정하고 자동 통합의 활성화 여부를 명시적으로 관리하는 것이 좋다.

## 보안 관점

- 자동 SDK 통합은 Bot Control 및 Fraud Control의 신호 수집을 돕지만 다른 보안 계층을 대체하지 않는다.
- Web ACL 규칙 우선순위와 Count, CAPTCHA, Challenge, Block 동작을 함께 검토한다.
- Content Security Policy(CSP)가 SDK 스크립트 로드를 허용하는지 AWS 문서의 지침에 따라 확인한다.
- HTML 변환이 애플리케이션의 보안 헤더, 캐시 동작, 페이지 렌더링에 미치는 영향을 사전 테스트한다.
- Web ACL 설정 변경 권한은 최소 권한 원칙에 따라 제한한다.
- AWS CloudTrail과 WAF 로그를 사용해 구성 변경과 요청 평가 결과를 추적한다.
- 수집되는 클라이언트 신호와 개인정보 처리 정책이 조직의 규정 준수 요구사항에 맞는지 검토한다.

자동 삽입이 활성화되었다고 해서 모든 봇과 사기가 자동으로 차단되는 것은 아니다. 관리형 규칙 그룹의 설정, 예외, 임계값과 실제 애플리케이션 트래픽을 함께 조정해야 한다.

## 운영 관점

기능을 활성화하기 전에 스테이징 CloudFront 배포에서 페이지와 보안 정책의 호환성을 확인하는 것이 좋다. 특히 강한 CSP를 사용하는 사이트는 스크립트 로드가 차단되지 않는지 검증해야 한다.

운영 시에는 다음 항목을 관리한다.

- CloudFront 배포와 Web ACL 연결 상태
- 대상 관리형 규칙 그룹 포함 여부
- 자동 SDK 통합 활성화 여부
- 콘솔 생성과 인프라 코드 생성 사이의 기본값 차이
- 기존 Web ACL의 명시적 마이그레이션 여부
- 브라우저 개발자 도구와 테스트를 통한 SDK 로드 확인
- WAF 로그, 샘플 요청, 관리형 규칙 레이블 및 지표
- 오탐과 미탐에 따른 규칙 조정
- CSP 및 애플리케이션 렌더링 호환성

자동 SDK 통합 자체에는 추가 요금이 없지만, 사용하는 AWS WAF 관리형 규칙 그룹과 요청 처리에는 기존 요금이 적용될 수 있다. 도입 전 현재 WAF 구성과 예상 트래픽을 기준으로 비용을 확인해야 한다.

## 시험 관점 정리

AWS 시험이나 아키텍처 문제에서는 다음 개념을 연결해서 기억할 수 있다.

- AWS WAF는 Web ACL과 규칙을 이용해 HTTP(S) 요청을 검사한다.
- CloudFront는 엣지에서 콘텐츠를 제공하며 AWS WAF Web ACL과 연결할 수 있다.
- 자동 SDK 통합은 HTML에 WAF JavaScript SDK 태그를 추가한다.
- Bot Control Targeted, ATP, ACFP가 클라이언트 측 신호를 사용한다.
- 콘솔 신규 생성과 프로그래밍 방식 신규 생성의 기본값이 다르다.
- 기존 Web ACL은 이번 출시로 자동 변경되지 않는다.
- CSP가 클라이언트 스크립트 통합에 영향을 줄 수 있다.
- 관리형 규칙도 로그와 지표를 기반으로 튜닝하고 예외를 관리해야 한다.

## 같이 보면 좋은 학습 포인트

- AWS WAF Web ACL, 규칙, 규칙 그룹의 구조
- AWS Managed Rules for Bot Control
- Fraud Control ATP와 ACFP의 차이
- CAPTCHA와 Challenge 동작
- Amazon CloudFront의 캐시 및 응답 처리 흐름
- Content Security Policy의 script-src 지시문
- AWS WAF 로깅과 CloudWatch 지표
- AWS Firewall Manager를 이용한 다중 계정 WAF 정책 관리

## 한 줄 정리

CloudFront용 AWS WAF 자동 SDK 통합으로 애플리케이션 코드를 수정하지 않고도 Bot Control과 Fraud Control에 필요한 클라이언트 신호를 더 일관되게 제공할 수 있게 되었다.
