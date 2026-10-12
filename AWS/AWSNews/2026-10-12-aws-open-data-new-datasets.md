# AWS Registry of Open Data에 46개 데이터셋 추가

- 정리 날짜: 2026-10-12
- 원문 게시일: 2026-10-11
- 공식 출처: [AWS Public Sector Blog - 46 new datasets available on the Registry of Open Data on AWS](https://aws.amazon.com/blogs/publicsector/46-new-datasets-available-on-the-registry-of-open-data-on-aws/)

## 핵심 내용

AWS가 Registry of Open Data on AWS에 새로 추가되거나 업데이트된 46개 공개 데이터셋을 소개했다. AWS Open Data Sponsorship Program은 공공 가치가 높은 데이터를 cloud-optimized 형태로 공개하고, 데이터 제공자가 부담해야 할 storage 비용을 AWS가 지원한다.

현재 이 프로그램을 통해 400PB가 넘는 데이터가 공개되어 있다. 데이터셋은 Registry of Open Data on AWS에서 검색할 수 있고 AWS Data Exchange에서도 발견할 수 있다. 사용자는 AWS에서 직접 데이터를 분석하거나 새로운 처리 기술, 데이터 형식과 도구를 개발할 수 있다.

이번 데이터셋은 climate and weather, geospatial, life sciences와 AI/ML 분야를 포함한다. 위성 및 기후 관측, 환경 위험, 농업 이미지, 유전체와 의료 영상처럼 용량이 크고 연구 활용도가 높은 데이터가 중심이다.

## 주요 데이터셋 사례

### Artemis와 Image Safari 농업 이미지

Alliance of Bioversity International과 CIAT가 제공한 농업 이미지 모음이다. 11개 국가와 18개 작물 종을 대상으로 740만 개가 넘는 field image, metadata와 annotation을 포함한다.

이 데이터는 agricultural foundation model, plant disease detection, growth stage recognition, crop identification와 농업 모니터링 연구에 활용할 수 있다. 특히 공개 데이터에서 상대적으로 부족했던 smallholder farming과 다양한 재배 환경을 보완한다.

### 기후 및 지리 공간 데이터

- NOAA의 Rapid Refresh Forecast System 및 ensemble forecast 데이터
- 남아메리카의 현재 및 미래 기후에 대한 고해상도 simulation
- NASA IMERG 기반 precipitation 데이터
- 캐나다 aerial imagery와 미국 전력망 데이터
- 수질, PFAS, lead service line과 환경 위험을 결합한 US Home Environmental Risk Compilation
- 해안, 대기와 지구 물리 관측 데이터

### 생명과학 및 AI 데이터

- neuroscience 및 brain connectivity microscopy 데이터
- prostate MRI와 precision imaging 데이터
- genomic, epigenomic 및 single-nucleus multiomics 데이터
- 병리학적 추론이 포함된 report generation 데이터
- 미국 National Archives가 제공하는 AI/ML 학습용 기록 데이터

## 왜 중요한가

대규모 공개 데이터는 다운로드와 복제만으로도 높은 storage 및 network 비용이 발생한다. 데이터가 AWS에 cloud-optimized 형태로 있으면 사용자는 전체 원본을 로컬로 옮기지 않고 데이터가 있는 곳에서 compute와 analytics workload를 실행할 수 있다.

연구자와 개발자는 동일한 원본을 공유해 실험 재현성을 높이고, 조직마다 같은 데이터를 중복 저장하는 비용을 줄일 수 있다. 공개된 metadata와 annotation은 machine learning 학습, benchmark와 foundation model 개발을 빠르게 시작하는 기반이 된다.

Common Crawl은 공개 데이터가 AI 생태계에 미치는 영향을 보여주는 대표 사례다. AWS는 2012년부터 Common Crawl의 web data repository를 지원해 왔으며, 이 데이터는 다수의 LLM과 연구 논문에서 활용됐다.

## 서비스 및 아키텍처 관점

1. 데이터 제공자가 고가치 데이터셋을 cloud-optimized 형식으로 준비한다.
2. Open Data Sponsorship Program이 공개 데이터의 AWS storage 비용을 지원한다.
3. 대용량 object와 partitioned dataset은 Amazon S3에 저장된다.
4. Registry of Open Data가 데이터 설명, license, resource 위치와 사용 방법을 catalog 형태로 제공한다.
5. AWS Data Exchange에서도 데이터셋을 검색하고 관련 정보를 확인할 수 있다.
6. 사용자는 데이터 전체를 내려받는 대신 AWS compute를 데이터와 가까운 위치에서 실행한다.
7. Amazon Athena, AWS Glue, Amazon EMR, Amazon SageMaker AI와 EC2 같은 서비스를 데이터 형식과 workload에 맞게 연결할 수 있다.

분석 아키텍처를 설계할 때는 원본 공개 bucket을 read-only source로 사용하고, 변환 결과와 model artifact는 사용자 계정의 별도 S3 bucket에 저장하는 방식이 적합하다. 대규모 객체를 매번 전체 scan하지 않도록 partition, columnar format와 metadata catalog를 활용해야 한다.

## 보안 및 운영 관점

- 공개 데이터라고 해도 dataset별 license, attribution과 허용된 사용 범위를 먼저 확인한다.
- 원본 bucket에는 write를 시도하지 않고 분석 결과는 소유한 S3 bucket에 분리한다.
- IAM role에는 필요한 S3 read와 분석 서비스 권한만 부여한다.
- requester-pays 또는 별도 egress 조건이 있는지 dataset documentation을 확인한다.
- 개인 정보, 의료 및 유전체 데이터는 공개 데이터 여부와 별개로 재식별 가능성과 조직의 규정 준수 기준을 검토한다.
- 데이터의 update 주기, version, provenance와 checksum을 기록해 분석 재현성을 확보한다.
- S3의 cross-Region data transfer와 대규모 query scan 비용을 예상하고 compute Region을 선택한다.
- Athena query에는 partition pruning과 column selection을 적용하고 workgroup budget 및 query limit을 설정한다.
- 변환 데이터에는 S3 encryption, versioning, lifecycle policy와 access logging을 적용한다.
- machine learning 학습 전에는 class imbalance, annotation 품질, geographic bias와 temporal drift를 평가한다.

## 비용 및 데이터 처리 관점

Open Data Sponsorship Program은 데이터 제공자의 공개 storage 비용을 지원하지만 사용자의 compute, query, 복사본 storage와 data transfer 비용까지 모두 무료라는 뜻은 아니다.

분석 비용을 줄이려면 데이터와 같은 Region에서 workload를 실행하고 필요한 object와 column만 읽어야 한다. 반복 사용 데이터는 적절한 형식으로 변환해 cache하되, 원본 전체를 무조건 복제하지 않는다. 실험이 끝난 임시 데이터와 intermediate artifact에는 S3 Lifecycle을 적용한다.

## 시험 관점 정리

- Amazon S3는 높은 durability와 확장성을 제공하는 object storage 서비스다.
- AWS Data Exchange는 third-party data product를 검색, 구독하고 사용하는 서비스다.
- AWS Glue Data Catalog는 dataset의 table 및 schema metadata를 중앙에서 관리한다.
- Amazon Athena는 S3 데이터를 표준 SQL로 조회하는 serverless query 서비스다.
- Amazon EMR은 Apache Spark, Hadoop 같은 big data framework를 managed cluster 또는 serverless 방식으로 실행한다.
- Amazon SageMaker AI는 data preparation, training와 inference workflow를 구축하는 managed ML 서비스다.
- 같은 Region에서 compute와 data를 함께 배치하면 latency와 cross-Region transfer 비용을 줄일 수 있다.
- columnar format, compression과 partitioning은 scan data를 줄여 analytics 성능과 비용을 개선한다.
- 공개 접근 가능 여부와 데이터 사용 license는 서로 다른 문제이므로 둘 다 확인해야 한다.

## 같이 보면 좋은 학습 포인트

- Registry of Open Data on AWS 사용 방법
- AWS Open Data Sponsorship Program 신청 기준
- Amazon S3 object storage와 bucket policy
- AWS Data Exchange와 public dataset catalog
- Athena partition projection 및 query cost optimization
- AWS Glue crawler와 Data Catalog
- EMR 및 SageMaker AI 기반 대규모 데이터 처리
- Parquet, Zarr와 cloud-optimized geospatial format
- ML dataset의 provenance, bias와 data quality
- S3 Lifecycle 및 Intelligent-Tiering

## 한 줄 정리

AWS Open Data Sponsorship Program은 46개의 기후, 지리 공간, 생명과학 및 AI 데이터셋을 추가해 사용자가 대규모 공개 데이터를 AWS에서 직접 분석하고 연구와 machine learning에 활용할 수 있게 했다.
