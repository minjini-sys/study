# AWS Lambda SnapStart, 컨테이너 이미지 함수 지원

- 정리 날짜: 2026-10-07
- 원문 게시일: 2026-10-06
- 공식 출처: [AWS Compute Blog - Get started with AWS Lambda SnapStart for container images](https://aws.amazon.com/blogs/compute/get-started-with-aws-lambda-snapstart-for-container-images/)

## 핵심 내용

AWS Lambda SnapStart가 컨테이너 이미지로 배포한 Lambda 함수를 지원한다. 컨테이너 이미지 함수는 조직의 컨테이너 배포 표준을 재사용하고 최대 10GB의 큰 의존성을 패키징할 수 있지만, 이미지 계층 다운로드와 런타임 초기화 때문에 cold start가 수 초 걸릴 수 있다.

SnapStart를 활성화하고 함수 버전을 게시하면 Lambda가 실행 환경을 미리 초기화한 뒤 전체 메모리와 디스크 상태를 Firecracker microVM snapshot으로 저장한다. 호출 시에는 컨테이너와 애플리케이션을 처음부터 초기화하지 않고 암호화되어 캐시된 snapshot을 복원해 시작 시간을 수 초에서 sub-second 수준까지 줄일 수 있다.

SnapStart는 게시된 함수 버전에만 적용되며 `$LATEST`에는 적용되지 않는다. 컨테이너 이미지는 Lambda 함수와 같은 AWS 리전의 Amazon ECR 저장소에 있어야 한다.

## 왜 중요한가

ML 추론, 대화형 API와 사용자 요청에 직접 연결된 함수는 cold start 지연이 사용자 경험에 영향을 준다. 컨테이너 이미지가 클수록 런타임과 프레임워크 초기화 비용이 커져 서버리스의 자동 확장 장점과 낮은 지연 시간 요구가 충돌할 수 있다.

SnapStart는 무거운 초기화 작업을 배포 시점에 한 번 수행하고 그 결과를 재사용한다. 이를 통해 컨테이너 기반 빌드 표준과 Lambda의 빠른 시작을 함께 사용할 수 있으며, 큰 라이브러리가 필요한 추론 및 API 워크로드의 응답 시간 편차를 줄일 수 있다.

## 동작 방식

1. 컨테이너 이미지를 빌드해 동일 리전의 Amazon ECR에 푸시한다.
2. `PackageType=Image`인 Lambda 함수에서 SnapStart를 `ApplyOn=PublishedVersions`로 설정한다.
3. 새 함수 버전을 게시하면 Lambda가 실행 환경을 만들고 초기화 코드를 실행한다.
4. 초기화된 microVM의 메모리와 디스크 상태를 암호화된 snapshot으로 생성하고 캐시한다.
5. 새로운 실행 환경이 필요할 때 Lambda는 이미지 초기화 대신 snapshot을 복원한다.
6. 해당 함수 버전을 삭제할 때까지 snapshot이 캐시되며 버전 삭제 시 관련 캐싱 비용도 중단된다.

기존 함수에서는 다음과 같이 SnapStart를 켜고 새 버전을 게시할 수 있다.

```bash
aws lambda update-function-configuration \
  --function-name my-function \
  --snap-start ApplyOn=PublishedVersions

aws lambda publish-version \
  --function-name my-function
```

## 지원 런타임과 이미지

출시 시 Lambda 제공 base image 기준 지원 범위는 다음과 같다.

- Java 11 이상: x86_64 및 arm64
- Python 3.12 이상: x86_64 및 arm64
- .NET 8 이상: x86_64 및 arm64

Java는 CRaC, Python은 `snapshot_restore_py`, .NET은 `SnapshotRestore`를 통해 snapshot 전과 restore 후 hook을 등록할 수 있다. Node.js, Ruby, custom Runtime Interface Client 또는 `provided.al2023` 기반 custom runtime은 `/restore/next` API를 구현하거나 hook이 필요하지 않은 경우 Dockerfile에 `com.amazonaws.lambda.feature.snapstart="Allow"` label을 추가해 opt-in할 수 있다.

## 서비스 및 아키텍처 관점

- **Amazon ECR**: Lambda 컨테이너 이미지를 저장하며 함수와 동일한 리전에 위치해야 한다.
- **AWS Lambda**: 게시 시 실행 환경을 초기화하고 snapshot을 생성하며 호출 시 복원한다.
- **Firecracker microVM**: 격리된 Lambda 실행 환경의 메모리와 디스크 상태를 snapshot으로 만든다.
- **Lambda version 및 alias**: SnapStart가 적용된 불변 버전을 alias에 연결해 트래픽을 안전하게 전환한다.
- **Amazon CloudWatch**: restore 시간, 실행 시간과 오류를 로그 및 지표로 관찰한다.
- **AWS X-Ray**: 함수 및 extension의 tracing과 end-to-end 지연 분석에 활용한다.

무거운 라이브러리 로딩, 모델 파일 준비와 클라이언트 생성은 가능한 경우 handler가 아니라 initialization code에서 수행한다. 그러면 해당 작업이 snapshot에 포함되어 매 restore마다 반복되지 않는다. 다만 연결, 자격 증명과 고유 값처럼 시간이 지나면 유효하지 않거나 실행 환경별로 달라야 하는 상태는 restore 후 다시 만들어야 한다.

## 보안 및 운영 관점

- 초기화 시 생성한 고유 ID, secret 또는 random value는 모든 복원 환경에 공유될 수 있으므로 handler나 after-restore hook에서 생성한다.
- Lambda는 restore 시 `/dev/random`과 `/dev/urandom`을 다시 seed하지만 애플리케이션 자체 난수 생성기의 상태는 별도로 점검한다.
- snapshot에 포함된 database connection은 만료될 수 있으므로 after-restore hook에서 재연결하거나 유효성을 검사한다.
- AWS SDK가 관리하는 네트워크 연결은 자동 복구되지만 직접 관리하는 연결에는 명시적인 복원 로직을 둔다.
- snapshot에 캐시된 임시 자격 증명, token과 timestamp가 만료되지 않았는지 요청 처리 전에 확인한다.
- 함수 version과 alias를 사용해 SnapStart 적용 버전을 명확히 배포하고 `$LATEST` 호출을 피한다.
- ECR 이미지 스캔, 서명과 최소 권한 실행 역할로 컨테이너 공급망을 보호한다.
- CloudWatch의 `RESTORE_REPORT`와 `REPORT` 항목에서 restore duration, billed restore duration과 실행 시간을 모니터링한다.
- 사용하지 않는 게시 버전을 삭제해 오래된 snapshot의 캐싱 비용과 공격 표면을 줄인다.

## 제한 사항과 비용

- Provisioned Concurrency와 함께 사용할 수 없다.
- Amazon EFS와 Amazon S3 Files는 지원되지 않는다.
- 512MB를 초과하는 ephemeral storage는 지원되지 않는다.
- 컨테이너 이미지의 최대 크기는 계속 10GB다.
- SnapStart snapshot을 캐시하는 시간과 snapshot에서 실행 환경을 복원할 때 비용이 발생한다.
- 아시아 태평양 뉴질랜드 및 타이베이를 제외한 모든 AWS 상용 리전에서 제공된다.

Provisioned Concurrency와 SnapStart는 모두 cold start를 줄이지만 방식과 비용 구조가 다르다. 항상 준비된 실행 환경이 필요한 일관된 초저지연 워크로드와 snapshot 복원으로 충분한 워크로드를 구분해 선택해야 한다.

## 시험 관점 정리

- Lambda cold start는 새 실행 환경 생성, 코드 또는 이미지 로드, 런타임 부팅과 initialization code 실행 과정에서 발생한다.
- SnapStart는 초기화된 실행 환경을 snapshot으로 저장하고 restore해 cold start를 줄인다.
- SnapStart는 게시된 version에 적용되며 `$LATEST`에는 적용되지 않는다.
- version은 불변이며 alias를 이용해 특정 version으로 트래픽을 라우팅할 수 있다.
- Lambda 컨테이너 이미지는 Amazon ECR에 저장하고 함수와 같은 리전에 두어야 한다.
- initialization code는 실행 환경마다 한 번 실행되지만 handler는 각 invocation을 처리한다.
- snapshot에서 복원되는 상태의 uniqueness와 freshness를 반드시 검토해야 한다.
- Provisioned Concurrency는 실행 환경을 미리 준비하고 SnapStart는 초기화 상태를 snapshot에서 복원한다.

## 같이 보면 좋은 학습 포인트

- Lambda execution environment와 cold start lifecycle
- Lambda version, alias 및 weighted routing
- SnapStart before-snapshot 및 after-restore hook
- Firecracker microVM의 격리 방식
- ECR 이미지 스캔과 container image signing
- Provisioned Concurrency와 SnapStart 비교
- CloudWatch `RESTORE_REPORT` 및 X-Ray tracing
- database connection pool과 temporary credential 갱신
- Java CRaC, Python `snapshot_restore_py`, .NET `SnapshotRestore`

## 한 줄 정리

Lambda SnapStart의 컨테이너 이미지 지원은 초기화된 microVM 상태를 snapshot으로 캐시하고 복원해 큰 이미지와 무거운 런타임의 cold start를 sub-second 수준까지 줄인다.
