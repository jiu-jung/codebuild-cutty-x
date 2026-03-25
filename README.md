# CodeBuild Repo README

## 개요

이 리포지토리는 **cutty-x FaaS 플랫폼의 빌드 엔진**으로, <br>
사용자가 업로드한 함수 코드를 **S3**에서 가져와 **Cloud Native Buildpacks 기반 컨테이너 이미지로 빌드**하고, <br>
완성된 이미지를 **Amazon ECR에 푸시**한 뒤 **후속 배포 단계로 전달**하는 파이프라인을 담당합니다. <br> <br>

단순히 이미지를 한 번 생성하는 빌드 스크립트가 아니라,<br>
서로 다른 사용자 코드가 들어오더라도 **같은 방식으로 준비·빌드·후속 전달까지 이어지는 표준 경로**를 만드는 데 초점을 맞췄습니다. <br>
이를 위해 CodeBuild 내부 로직을 단계별로 분리하고, 사용자 코드의 실행 편차를 사전에 보정하며, 성공·실패 결과가 다음 시스템으로 바로 전달되도록 구성했습니다. <br> <br>

특히 이 프로젝트에서는 다음 방향에 집중했습니다.
- **pre_build / build / post_build 단계 분리**를 통해 책임과 장애 지점을 명확히 구조화
- 사용자 코드 차이를 흡수하기 위한 **`package.json` 보정, 의존성 주입, `.env` 생성, 엔트리포인트 래핑 자동화**
- 빌드 성공 시 **이미지 다이제스트 기반 메시지 전파**, 실패 시 **즉시 실패 알림 전파**
- CodeBuild 재실행 비용을 줄이기 위한 **pack CLI / Docker layer / volume 캐시** 적용

<br>

<p align="center">
    <img src=./diagram.png width="1000px"/>
</p>


<br>

---

<br>

## 설계 포인트
#### 1. 빌드 과정을 단계별로 나눠 장애 지점과 변경 범위를 분리

`buildspec.yml`은 **install → pre_build → build → post_build** 순서로 실행되며, <br>
실제 스크립트도 `01_prebuild`, `02_build`, `03_postbuild` 디렉터리로 나누어 관리했습니다. <br>
덕분에 “환경 준비”, “이미지 생성”, “후속 시스템 전달”이 섞이지 않고, 문제가 발생했을 때 어느 단계에서 실패했는지 바로 좁힐 수 있습니다. <br>

또한 공통 실패 처리 로직은 `common.sh`의 `notify_deploy_failed()`로 묶어, <br>
**각 단계가 독립적으로 동작하면서도 실패 상태는 같은 형식으로 전파**되도록 정리했습니다.

<br>

#### 2. 사용자 코드의 편차를 빌드 엔진 내부에서 흡수

이 파이프라인은 사용자 코드를 그대로 빌드에 넘기지 않고, pre-build 단계에서 먼저 실행 가능한 형태로 정리합니다. <br>
S3에서 함수 코드를 동기화한 뒤 `index.js` 존재 여부를 확인하고, Node 문법 검사까지 수행해 잘못된 코드는 빌드 전에 차단합니다. `package.json`이 없으면 기본 골격을 생성하고, 있더라도 플랫폼 실행에 필요한 의존성과 start 스크립트를 강제로 보정합니다. <br>

또한 `CUSTOM_ENV` 값을 JSON 배열 형식으로 받아 `.env` 파일로 변환하고, <br>
필요한 경우 기존 `index.js`를 `index-original.js`로 바꾼 뒤 새 wrapper를 생성해 `dotenv`를 먼저 로드하도록 구성했습니다. <br>
이 wrapper는 요청 헤더의 `X-Request-ID`를 로그에 붙이고, 실행 종료 시 성공·실패 상태를 내부 로그로 남기도록 되어 있습니다. <br>

즉, 사용자는 함수 코드만 제출해도 되고, <br>
빌드 엔진이 그 코드를 **플랫폼 표준 실행 경로에 맞는 컨테이너 입력 형태로 보정**하는 구조입니다.

<br>

#### 3. 이미지를 만드는 데서 끝내지 않고 다음 배포 단계까지 연결

build 단계에서는 `pack build`로 이미지를 생성하고 Amazon ECR에 푸시합니다. <br>
그 다음 post-build 단계에서는 단순 성공 로그만 남기는 것이 아니라, ECR에서 **태그 기준 이미지 다이제스트를 조회**한 뒤, 이를 포함한 메시지를 **SQS**로 전송합니다. 메시지에는 `userId`, `functionId`, `ImageDigest`, `customRoutes`가 담기며, 다이어그램상 이 메시지는 queue polling agent가 받아 **Knative Service CRD** 생성으로 이어집니다. <br>

실패 시에는 각 단계에서 `notify_deploy_failed()`를 호출해 Amplify 쪽 실패 webhook으로 상태를 전달하도록 했습니다. <br>
즉, 이 파이프라인은 단순한 이미지 빌드 작업이 아니라, **프론트엔드 트리거부터 런타임 반영까지 이어지는 배포 체인의 한 단계**로 설계되어 있습니다.

<br>

---

<br>

## 디렉터리 구조

빌드 스크립트는 실행 단계와 책임에 따라 아래와 같이 분리되어 있습니다.
```bash
.
├── buildspec.yml                    # AWS CodeBuild 빌드 명세
└── scripts
    ├── 01_prebuild/
    │   ├── install_pack.sh          # pack CLI 설치
    │   ├── resolve_env.sh           # AWS / S3 / ECR / SQS 관련 환경 계산 및 ECR 로그인
    │   ├── s3_sync_and_validate.sh  # 사용자 코드 S3 동기화, index.js 검증, 문법 검사
    │   ├── patch_package_json.sh    # package.json 보정 스크립트 호출
    │   └── process_env_files.sh     # CUSTOM_ENV를 .env로 변환하고 wrapper 생성
    ├── 02_build/
    │   └── build_image.sh           # pack build 실행 및 ECR 푸시
    ├── 03_postbuild/
    │   └── publish_sqs_message.sh   # 이미지 다이제스트 조회 후 SQS 메시지 전송
    ├── lib/
    │   ├── create_env_file.py       # .env 파일 생성
    │   ├── create_env_wrapper.py    # dotenv 로딩용 index.js wrapper 생성
    │   ├── parse_custom_env.py      # CUSTOM_ENV를 pack --env 플래그로 바꾸는 대안 유틸
    │   └── patch_package_json.py    # 의존성 및 start 스크립트 보정
    └── common.sh                    # 공통 환경 로드 및 실패 알림 함수
```

<br>

---

<br>

## 빌드 파이프라인
`buildspec.yml`은 4단계 라이프사이클로 구성되어 있습니다.

#### 1. Install
빌드에 필요한 공통 도구를 준비합니다.
- **install_pack.sh**
    - `pack CLI`를 설치합니다.
    - `/usr/local/bin/pack`에 파일이 이미 있으면 캐시를 재사용합니다.
- **로컬 캐시**
    - `LOCAL_DOCKER_LAYER_CACHE`
    - `LOCAL_SOURCE_CACHE`
    - `LOCAL_CUSTOM_CACHE`
    - 커스텀 캐시 경로: `/usr/local/bin/pack`, `/var/lib/docker/volumes/**/*`

이 단계는 매 빌드마다 동일한 도구를 다시 내려받지 않도록 해 반복 실행 비용을 줄이는 역할을 합니다.

<br>

#### 2. Pre_build Phase
실제 빌드 전에 사용자 코드를 플랫폼이 처리 가능한 형태로 정리하는 단계입니다.
- **resolve_env.sh**
    - AWS Account ID와 Region을 확인합니다.
    - `S3_URL`, `IMAGE_REPO`, `IMAGE_TAG_FULL`, `IMAGE` 값을 계산합니다.
    - 계산된 값을 `/tmp/build.env`에 저장하고 ECR 로그인까지 수행합니다.
- **s3_sync_and_validate.sh**
    - `s3://{USER_CODE_BUCKET}/users/{USER_ID}/functions/{FUNCTION_ID}` 경로에서 코드를 가져옵니다.
    - `index.js` 존재 여부를 확인합니다.
    - `node -c ./app/index.js`로 JavaScript 문법 검사까지 수행합니다.
- **patch_package_json.sh**
    - `patch_package_json.py`를 호출합니다.
    - `@google-cloud/functions-framework`와 `dotenv` 의존성을 보정합니다.
    - `functions-framework --target=handler --port=${PORT:-8080} --host=0.0.0.0` 형태의 start 스크립트를 보장합니다.
    - `package.json`이 없으면 기본 skeleton을 생성합니다.
- **process_env_files.sh**
    - `CUSTOM_ENV`를 `.env` 파일로 변환합니다.
    - `.env`가 생성되면 `index.js`를 wrapper 구조로 교체해 런타임에서 환경 변수를 먼저 로드하도록 만듭니다.

이 단계의 핵심은
사용자 코드마다 다른 실행 편차를 사전에 흡수해 **빌드 실패와 런타임 불일치를 줄이는 것**입니다.

<br>

#### 3. Build Phase

준비된 소스를 실제 컨테이너 이미지로 만드는 단계입니다.

- **build_image.sh**
    - `pack build "${IMAGE}"` 실행
    - builder: `gcr.io/buildpacks/builder:latest`
    - source path: `./app`
    - 빌드 성공 후 `docker push "${IMAGE}"`로 Amazon ECR에 푸시

이 단계에서 Dockerfile을 직접 받지 않고도
Cloud Native Buildpacks를 통해 **일관된 컨테이너 생성 경로**를 제공할 수 있습니다.

<br>

#### 4. Post_build Phase
빌드 결과를 후속 시스템으로 전달하는 단계입니다.
- **publish_sqs_message.sh**
    - ECR에서 태그 기준 이미지 다이제스트를 조회합니다.
    - `IMAGE_REPO@IMAGE_DIGES`T 형식의 digest URI를 생성합니다.
    - `{userId, functionId, ImageDigest, customRoutes}` 메시지를 SQS로 전송합니다.
- **실패 처리**
    - 필수 ECR / SQS 환경 값이 없거나, 다이제스트 조회 또는 SQS 전송에 실패하면 즉시 실패 상태를 전파합니다.
    - 실패 알림은 `common.sh`의 webhook 호출 로직을 통해 Amplify 쪽으로 전달됩니다.

즉, post-build는 단순 마무리 단계가 아니라, **빌드 결과를 다음 배포 시스템이 소비할 수 있는 배포 이벤트로 변환**하는 역할을 합니다.

<br>

---

<br>


## Python 유틸리티 스크립트 (`scripts/lib/`)

복잡한 파일 조작과 JSON 처리 로직은 Python으로 분리해,
쉘 스크립트는 단계 실행과 흐름 제어에 집중하도록 구성했습니다.

- **patch_package_json.py** <br>
    `package.json`을 읽거나 새로 만들고, `@google-cloud/functions-framework`, `dotenv` 의존성과 start 스크립트를 보장합니다.
- **create_env_file.py** <br>
    `CUSTOM_ENV`를 JSON 배열로 파싱해 `.env` 파일로 변환합니다. 유효하지 않은 key/value는 건너뛰고, 값에 공백이나 특수문자가 있으면 적절히 이스케이프합니다.
- **create_env_wrapper.py** <br>
    기존 `index.js`를 `index-original.js`로 변경한 뒤, `.env`를 먼저 로드하고 요청 단위 로그를 감쌀 수 있는 새 `index.js` wrapper를 생성합니다.
- **parse_custom_env.py** <br>
    `CUSTOM_ENV`를 `pack build --env` 플래그 문자열로 바꾸는 대안 유틸입니다. 현재 기본 buildspec 경로에서는 직접 사용되지 않지만, 환경 변수 주입 방식을 바꿀 수 있도록 남겨둔 확장 포인트입니다.

<br>

---

<br>


## 캐시 설정
빌드 시간을 줄이고 재실행 효율을 높이기 위해 CodeBuild 로컬 캐시를 사용합니다.

- **Source Cache** <br> 다운로드된 소스와 관련 파일 재사용
- **Docker Layer Cache** <br> 이전 이미지 빌드 레이어 재사용
- **Custom Cache** <br> `pack CLI` 및 buildpacks 관련 Docker volume 캐시
    - `/usr/local/bin/pack`
    - `/var/lib/docker/volumes/**/*`

이 캐시 구성 덕분에 동일한 도구 설치와 빌드 준비를 반복하는 비용을 줄일 수 있습니다.
<br>

---

<br>

## 기대 효과

이 CodeBuild 파이프라인은 다음 특성을 갖습니다.
- 사용자 코드 차이를 내부에서 흡수하는 **표준 빌드 구조**
- 단계별 책임 분리로 장애 지점을 추적하기 쉬운 **운영 친화적 구조**
- 실패 시 즉시 상태를 전파하는 **가시성 있는 배포 흐름**
- 빌드 결과를 이미지 다이제스트 기반 이벤트로 전달하는 **확장 가능한 후속 배포 인터페이스**

즉, 이 프로젝트의 CodeBuild 구성은 단순한 빌드 스크립트가 아니라, <br>
FaaS 플랫폼에서 **다양한 함수 코드를 같은 방식으로 준비하고, 컨테이너로 빌드하고, 다음 런타임 단계까지 연결하는 빌드 오케스트레이션 레이어**입니다.
