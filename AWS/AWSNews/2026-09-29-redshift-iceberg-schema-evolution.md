# Amazon Redshift의 Apache Iceberg 스키마 및 파티션 진화 지원

- 정리 날짜: 2026-09-29
- 원문 게시일: 2026-09-28
- 공식 출처: [AWS Big Data Blog - Getting started with Apache Iceberg write support in Amazon Redshift Part 3](https://aws.amazon.com/blogs/big-data/getting-started-with-apache-iceberg-write-support-in-amazon-redshift-part-3/)

## 핵심 내용

Amazon Redshift에서 Apache Iceberg 테이블의 스키마와 파티션 구성을 `ALTER TABLE` 문으로 변경할 수 있다. 열 추가·삭제·이름 변경, 호환되는 데이터 형식 확장, 향후 쓰기의 압축 방식 변경과 파티션 필드 추가·삭제·교체가 지원된다.

대부분의 변경은 기존 데이터 파일을 다시 작성하지 않고 Iceberg 메타데이터만 수정한다. 기존 파일은 원래 위치와 구조를 유지하고 새로 기록되는 데이터부터 변경된 스키마나 파티션 규칙을 사용한다. 쿼리 엔진은 이전 구조와 새로운 구조를 함께 읽는다.

AWS Lake Formation 리소스 링크를 Amazon S3 Tables 카탈로그에 생성하면 Redshift, Athena, EMR과 다른 분석 엔진이 동일한 Iceberg 테이블을 중앙 권한 모델 아래에서 검색하고 사용할 수 있다.

## 왜 중요한가

운영 데이터의 열과 데이터 형식은 계속 바뀌고 쿼리 패턴에 맞춰 파티션 전략도 조정된다. 전통적인 데이터 레이크에서는 이런 변경 때문에 대량의 데이터를 다시 쓰거나 ETL 파이프라인을 재구축해야 했다.

Iceberg의 메타데이터 기반 진화를 사용하면 데이터 규모에 비례하는 재작성 작업 없이 테이블 구조를 바꿀 수 있다. 스키마 변경이 Redshift 외의 엔진에도 자동으로 보이므로 하나의 데이터 복사본을 여러 분석 도구가 공유하는 레이크하우스 구성이 쉬워진다.

## 지원되는 주요 작업

- `ADD COLUMN`, `DROP COLUMN`: 열을 추가하거나 현재 스키마에서 제거한다.
- `RENAME COLUMN`: 데이터와 파티션 구성을 바꾸지 않고 열 이름을 변경한다.
- `ALTER COLUMN TYPE`: `INT`에서 `BIGINT`처럼 안전한 범위 확장을 수행한다.
- `SET TABLE PROPERTIES`: 이후 기록되는 파일의 압축 방식 등을 변경한다.
- `ADD/DROP/REPLACE PARTITION FIELD`: 데이터 재작성 없이 파티션 규칙을 진화시킨다.
- Lake Formation resource link: S3 Tables를 Glue Data Catalog와 연결해 엔진 간 접근을 중앙 관리한다.

## 서비스 및 아키텍처 관점

1. Amazon Redshift Provisioned 또는 Serverless가 Iceberg 테이블에 SQL 쓰기 작업을 수행한다.
2. 테이블 데이터와 메타데이터는 일반 S3 버킷 또는 Amazon S3 Tables에 저장된다.
3. AWS Glue Data Catalog가 테이블 정의와 카탈로그 정보를 관리한다.
4. `ALTER TABLE` 실행 시 Iceberg 메타데이터가 갱신된다.
5. Lake Formation 리소스 링크가 S3 Tables 카탈로그를 기본 Glue Data Catalog에 연결한다.
6. Redshift, Athena, EMR과 BI 애플리케이션이 중앙 권한에 따라 같은 테이블을 조회한다.

S3 Tables를 직접 참조하는 3단계 표기법은 IAM 연동 자격 증명이 필요하다. 데이터베이스 사용자, BI 도구, JDBC/ODBC 애플리케이션에는 명시적인 IAM 역할을 가진 외부 스키마를 사용할 수 있다.

## 파티션 진화에서 주의할 점

파티션을 일 단위에서 월 단위로 변경해도 기존 파일은 일 단위 폴더에 남고 새 파일은 월 단위 규칙을 따른다. Iceberg는 두 레이아웃을 투명하게 읽지만 운영자는 여러 파티션 규칙이 공존한다는 점을 알고 있어야 한다.

시간 범위 쿼리에는 `year`, `month`, `day`, `hour` 변환을 사용할 수 있고, 값의 종류가 많은 조인 키에는 `bucket`을 고려할 수 있다. 현재 파티션에 사용되는 열을 삭제하려면 먼저 해당 파티션 필드를 삭제하거나 교체해야 한다.

## 보안 및 운영 관점

- Lake Formation에서 테이블, 열과 행 수준의 접근 권한을 최소 권한으로 부여한다.
- 외부 스키마 권한을 `PUBLIC`에 주지 않고 지정된 IAM 역할이나 데이터베이스 사용자에게만 제공한다.
- 메타데이터 변경도 모든 리더에게 즉시 보이므로 운영 적용 전 비운영 환경에서 검증한다.
- `DROP`과 `ADD`를 따로 실행하기보다 원자적인 `REPLACE PARTITION FIELD`를 우선 사용한다.
- `SHOW TABLE`로 변경 후 스키마와 파티션 명세를 확인한다.
- 여러 `UPDATE`, `DELETE`, `MERGE` 후 AWS Glue Table Optimizer로 삭제 파일과 작은 파일을 정리한다.
- Iceberg 테이블 삭제는 카탈로그 항목만 제거할 수 있으므로 남은 S3 데이터와 비용을 별도로 점검한다.

## 시험 관점 정리

- Amazon Redshift는 데이터 웨어하우스이며 Spectrum과 외부 스키마를 통해 데이터 레이크를 조회할 수 있다.
- Apache Iceberg는 ACID 트랜잭션, 스키마 진화, 파티션 진화와 시간 여행을 지원하는 오픈 테이블 형식이다.
- AWS Glue Data Catalog는 여러 분석 서비스가 공유하는 메타데이터 카탈로그 역할을 한다.
- AWS Lake Formation은 데이터 레이크의 중앙 권한과 세분화된 접근 제어를 제공한다.
- Amazon S3 Tables는 Iceberg 테이블 저장 및 관리를 단순화하는 S3 기능이다.
- IAM 역할과 Lake Formation 권한은 함께 검토해야 실제 데이터 접근이 가능하다.

## 같이 보면 좋은 학습 포인트

- Apache Iceberg의 snapshot, manifest와 field ID
- 스키마 진화와 데이터 형식 확장 규칙
- partition pruning과 hidden partitioning
- Amazon Redshift 외부 스키마와 Spectrum
- S3 Tables와 일반 S3 기반 Iceberg 테이블의 차이
- Lake Formation resource link와 교차 계정 공유
- AWS Glue Table Optimizer의 compaction 및 orphan file 정리

## 한 줄 정리

Amazon Redshift의 Iceberg 쓰기 지원은 데이터 파일을 다시 작성하지 않고 스키마와 파티션을 진화시키고, Lake Formation을 통해 여러 분석 엔진의 접근을 중앙에서 관리할 수 있게 한다.
