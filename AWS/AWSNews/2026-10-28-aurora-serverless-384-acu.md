# Amazon Aurora Serverless, 최대 384 ACU 지원

## 공식 출처

- [Amazon Aurora serverless now supports up to 384 ACUs](https://aws.amazon.com/about-aws/whats-new/2026/10/aurora-serverless-supports-384-acu/)
- 발표일: 2026년 10월 27일

## 핵심 내용

Amazon Aurora Serverless의 Aurora PostgreSQL-Compatible Edition이 데이터베이스당 최대 384 Aurora Capacity Units(ACU)를 지원한다. 기존 최대값인 256 ACU에서 50% 증가한 규모다.

ACU는 Aurora Serverless의 처리 용량 단위다. AWS 발표에 따르면 ACU 1개는 약 2 GiB의 메모리와 그에 상응하는 CPU 및 네트워크 자원을 제공한다. 따라서 최대 384 ACU를 설정하면 단일 Aurora Serverless 데이터베이스가 최대 약 768 GiB의 메모리 규모까지 자동으로 확장할 수 있다.

384 ACU를 사용하려면 Platform Version 4와 다음 Aurora PostgreSQL 버전 중 하나가 필요하다.

- Aurora PostgreSQL 18.4
- Aurora PostgreSQL 17.10
- Aurora PostgreSQL 16.14
- Aurora PostgreSQL 15.18
- Aurora PostgreSQL 14.23

## 왜 중요한가

이전에는 최대 256 ACU를 넘는 순간 Aurora Serverless의 자동 확장 범위만으로 처리하기 어려운 워크로드가 있었다. 최대 용량이 384 ACU로 증가하면서 트래픽 변화가 크고 순간적으로 많은 메모리와 연산 능력이 필요한 데이터베이스도 서버리스 방식으로 운영할 수 있는 범위가 넓어졌다.

특히 에이전틱 AI 애플리케이션은 다음과 같은 특성을 보일 수 있다.

- 여러 에이전트가 동시에 작업할 때 부하가 급증한다.
- 실행 사이에는 긴 유휴 시간이 발생한다.
- 사용자 요청과 도구 호출 수에 따라 트래픽 예측이 어렵다.
- 대화 상태, 작업 기록, 검색 결과를 관계형 데이터베이스에 저장할 수 있다.

Aurora Serverless는 수요에 맞춰 확장하고, 워크로드가 끝난 뒤에는 0 ACU까지 축소할 수 있다. 따라서 높은 최대 용량을 확보하면서도 실제 사용한 용량을 기준으로 비용을 관리할 수 있다.

## 서비스 및 아키텍처 관점

384 ACU는 데이터베이스의 항상 고정된 크기가 아니라 자동 확장의 상한값이다. 최소 및 최대 ACU 범위를 워크로드에 맞게 설정하고, Aurora Serverless가 그 범위 안에서 용량을 조절하도록 구성한다.

신규 클러스터는 지원되는 Aurora PostgreSQL 엔진 버전과 Platform Version 4를 선택한 뒤 최대 용량을 384 ACU로 설정하면 된다.

기존 클러스터는 플랫폼 버전에 따라 절차가 달라진다.

1. 현재 엔진 버전과 Platform Version을 확인한다.
2. 이미 Platform Version 4라면 최대 ACU 설정을 변경한다.
3. 필요하면 지원되는 Aurora PostgreSQL 엔진 버전으로 업그레이드한다.
4. 이전 플랫폼 버전이라면 pending maintenance action을 통해 Platform Version 4로 업그레이드한다.
5. 변경 후 부하 테스트와 모니터링으로 확장 동작을 검증한다.

AWS 발표에 따르면 데이터베이스가 이미 Platform Version 4를 사용 중이라면 최대 용량 변경에 따른 중단이 없다. 다만 엔진 또는 플랫폼 업그레이드가 필요한 환경은 유지 관리 일정과 애플리케이션 호환성을 별도로 검토해야 한다.

아키텍처에서는 최대 ACU만 높이는 것으로 모든 병목이 해결된다고 가정하면 안 된다. 애플리케이션 연결 수, 쿼리 효율, 인덱스, 잠금, 커넥션 풀, 읽기 확장 방식도 함께 확인해야 한다.

## 보안 관점

- 클러스터 생성과 수정 권한은 필요한 운영 역할에만 최소 권한으로 부여한다.
- AWS CloudTrail을 통해 클러스터 용량 및 엔진 설정 변경 이력을 감사한다.
- 저장 데이터는 AWS KMS를 이용한 암호화를 적용하고 키 정책을 점검한다.
- 전송 구간에는 TLS를 사용하고 보안 그룹으로 데이터베이스 접근 경로를 제한한다.
- Secrets Manager 등으로 데이터베이스 자격 증명을 관리하고 정기적으로 교체한다.
- 엔진 업그레이드 전 보안 패치와 확장 기능의 호환성을 확인한다.

## 운영 관점

최대 ACU 증가는 더 큰 부하를 처리할 여유를 제공하지만, 실제 확장량이 증가하면 비용도 함께 늘어날 수 있다. 따라서 단순히 상한을 384 ACU로 설정하기보다 관측 지표와 부하 테스트를 근거로 결정해야 한다.

운영 시에는 다음 항목을 확인한다.

- 현재 엔진 버전과 Platform Version 4 지원 여부
- 최소 및 최대 ACU 설정
- ACU 사용량과 확장·축소 패턴
- CPU, 메모리 압력, 연결 수, 지연 시간, 쿼리 처리량
- 0 ACU 축소 사용 시 재활성화 지연이 업무 요구사항에 미치는 영향
- 엔진 및 플랫폼 업그레이드의 유지 관리 일정
- 최대 용량 증가에 따른 예상 비용과 비용 알림

높은 부하가 반복된다면 쿼리 튜닝, 읽기 복제본, 캐시, 비동기 처리 같은 아키텍처 개선도 함께 비교해야 한다. 384 ACU는 확장 가능한 선택지를 늘려 주지만 비효율적인 쿼리나 연결 관리 문제를 대신 해결하지는 않는다.

## 시험 관점 정리

AWS 시험이나 아키텍처 문제에서는 다음 개념을 연결해서 기억할 수 있다.

- Aurora Serverless는 ACU를 기준으로 데이터베이스 용량을 자동 조절한다.
- ACU 1개는 약 2 GiB 메모리와 그에 상응하는 CPU 및 네트워크 자원을 제공한다.
- 최대 384 ACU는 약 768 GiB 메모리 규모에 해당한다.
- 이번 지원 대상은 Aurora PostgreSQL-Compatible Edition이다.
- 384 ACU를 사용하려면 지원 엔진 버전과 Platform Version 4가 필요하다.
- 기존 클러스터가 Platform Version 4라면 최대 용량 설정을 중단 없이 변경할 수 있다.
- 서버리스 자동 확장도 최대 용량, 비용, 연결, 쿼리 성능에 대한 모니터링이 필요하다.
- 예측하기 어려운 간헐적 트래픽은 서버리스 데이터베이스가 적합한 대표 사례다.

## 같이 보면 좋은 학습 포인트

- Aurora Serverless의 최소·최대 ACU 설정
- Aurora PostgreSQL 엔진 버전과 Platform Version
- Aurora Auto Scaling과 읽기 복제본의 차이
- RDS Proxy를 이용한 데이터베이스 연결 관리
- Performance Insights와 CloudWatch 데이터베이스 지표
- Aurora의 백업, 복구, 암호화, 고가용성 구조
- 관계형 데이터베이스 비용 최적화 방법

## 한 줄 정리

Aurora PostgreSQL Serverless가 Platform Version 4에서 최대 384 ACU까지 확장되면서, 변동성이 크고 순간 부하가 높은 워크로드를 서버리스 방식으로 처리할 수 있는 범위가 넓어졌다.
