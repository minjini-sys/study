# Amazon EMR on EC2에서 Spark Connect 지원

- 정리 날짜: 2026-09-24
- 원문 게시일: 2026-09-23
- 공식 출처: [AWS Big Data Blog - Announcing Spark Connect on Amazon EMR on EC2: Interactive PySpark anywhere](https://aws.amazon.com/blogs/big-data/announcing-spark-connect-on-amazon-emr-on-ec2-interactive-pyspark-anywhere/)

## 핵심 내용

AWS가 Amazon EMR on EC2에서 Spark Connect를 지원한다고 발표했다. AWS Runtime for Apache Spark의 `emr-spark-8.0`, Apache Spark 4.0.2 이상에서 사용할 수 있다.

개발자는 Amazon SageMaker Unified Studio Data Notebook뿐 아니라 Visual Studio Code, PyCharm, Kiro, Jupyter 같은 로컬 개발 환경에서 PySpark 코드를 작성하고 디버깅할 수 있다. Python 코드는 로컬에서 실행되지만 DataFrame과 SQL 연산은 전용 EMR 클러스터의 Spark 엔진이 처리한다.

각 Spark Connect 세션은 독립된 실행 역할, 태그, 수명 주기를 갖는다. 따라서 여러 사용자가 하나의 클러스터를 공유하면서도 데이터 접근 권한과 비용을 세션별로 구분할 수 있다.

## 왜 중요한가

기존 EMR 개발에서는 클러스터에 연결된 Notebook을 사용하거나 코드를 작업으로 패키징해 제출한 뒤 결과를 확인해야 했다. 로컬 환경과 클러스터의 Spark 버전 및 라이브러리가 다르면 개발 중에는 보이지 않던 문제가 운영 환경에서 발생하기도 했다.

Spark Connect는 로컬 개발 편의성과 운영 클러스터의 실제 실행 환경을 연결한다. 개발자는 중단점을 설정하고 변수를 살펴보면서 전체 크기의 데이터를 처리할 수 있고, 검증한 코드를 동일한 클러스터에서 배치 작업으로 실행할 수 있다.

## 서비스 및 아키텍처 관점

Spark Connect는 클라이언트와 Spark 엔진을 분리한 구조다.

1. 로컬 IDE나 SageMaker Unified Studio에서 경량 PySpark 클라이언트를 실행한다.
2. Amazon EMR이 클러스터에서 Spark Connect Server를 YARN 애플리케이션으로 시작한다.
3. EMR이 세션 엔드포인트와 수명이 짧은 인증 토큰을 발급한다.
4. 클라이언트가 gRPC/TLS 연결로 DataFrame과 SQL 연산 계획을 서버에 보낸다.
5. 클러스터의 Spark 엔진이 연산을 수행하고 결과를 로컬 세션으로 반환한다.

하나의 클러스터는 서비스 한도 기준 최대 1,000개의 동시 세션과 1,000개의 동시 실행 역할을 지원한다. 실제 동시 처리 능력은 클러스터 크기와 세션별 작업량에 따라 달라지며, Amazon EMR Managed Scaling으로 수요에 맞게 용량을 조절할 수 있다.

## 보안 및 운영 관점

- 세션별 실행 역할에 최소 권한을 적용해 사용자가 필요한 데이터에만 접근하도록 한다.
- 세션 생성 권한과 `iam:PassRole` 권한을 엄격하게 제한한다.
- 전송 구간은 gRPC/TLS를 사용하고 반환된 단기 인증 토큰을 코드나 저장소에 기록하지 않는다.
- 프라이빗 서브넷에서는 SageMaker Unified Studio와 세션 엔드포인트 사이의 네트워크 경로를 확인한다.
- 실행 역할과 태그를 이용해 사용자 또는 팀별 사용량과 비용을 추적한다.
- 로컬 연결을 닫는 `spark.stop()`만으로 원격 세션은 종료되지 않으므로 작업 후 세션과 클러스터를 명시적으로 종료한다.

## 시험 관점 정리

- Amazon EMR은 Apache Spark 같은 빅데이터 프레임워크를 AWS에서 실행하는 관리형 서비스다.
- Spark Connect는 클라이언트와 서버를 분리하고 원격 Spark 엔진에서 연산을 수행한다.
- IAM 실행 역할은 세션별 AWS 리소스 접근 권한을 결정한다.
- EMR Managed Scaling은 작업 수요에 맞춰 클러스터 용량을 조정한다.
- 프라이빗 서브넷의 서비스 연결에는 VPC 엔드포인트와 네트워크 권한을 함께 고려한다.
- 비용 최적화에서는 공유 클러스터, 세션 종료, Spot 또는 Savings Plans 선택을 검토한다.

## 같이 보면 좋은 학습 포인트

- Apache Spark의 Driver, Executor, YARN 구조
- Spark Connect의 gRPC 기반 클라이언트-서버 통신
- Amazon EMR Runtime Role과 `iam:PassRole`
- Amazon SageMaker Unified Studio Data Notebook
- AWS Glue Data Catalog와 Apache Iceberg 연동
- EMR Managed Scaling 및 Spark History Server

## 알아둘 제한 사항

- 로컬 PySpark 버전은 EMR 클러스터의 Spark 버전과 일치해야 한다.
- DataFrame과 SQL API를 지원하지만 RDD 기반 API는 지원하지 않는다.
- 인증 토큰은 1시간 후 만료된다.
- 세션의 기본 유휴 시간 제한은 60분이며 최대 24시간까지 설정할 수 있다.
- 이번 릴리스에서는 다중 Primary Node 고가용성 클러스터, Trusted Identity Propagation, AWS Lake Formation 세분화 접근 제어를 지원하지 않는다.

## 한 줄 정리

Spark Connect on Amazon EMR on EC2는 로컬 IDE의 디버깅 경험과 운영 규모의 Spark 클러스터를 연결해 PySpark 개발 속도와 실행 환경의 일관성을 높인다.
