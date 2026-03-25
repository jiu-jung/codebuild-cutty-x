# CodeBuild Repo README

## 개요

이 리포지토리는 **cutty-x FaaS 플랫폼의 빌드 엔진**으로, <br>
사용자 코드를 **S3**에서 가져와 **Cloud Native Buildpacks로 컨테이너 이미지화**하고, <br>
이를 **Amazon ECR에 푸시한 뒤 빌드 결과를 시스템에 전파**하는 전체 파이프라인을 담당합니다. <br> <br>

단순히 이미지를 빌드하는 수준이 아니라, 다양한 사용자 코드가 들어오는 환경에서도 <br>
**일관된 방식으로 빌드·배포·후속 처리까지 연결되는 표준 파이프라인**이 되도록 설계했습니다. <br> <br>

특히 이 저장소에서는 다음과 같은 방향에 집중했습니다.
- **빌드 단계를 pre/build/post로 분리**해 책임을 명확히 구조화
- 사용자 코드 차이를 흡수하기 위해 **의존성 및 실행 진입점 자동 보정**
- 빌드 성공/실패 결과를 외부 시스템으로 즉시 전달해 **운영 대응 가능성 확보**
- 캐시와 모듈화를 통해 **재사용성과 유지보수성 개선**

<br>

<p align="center">
    <img src=./diagram.png width="1000px"/>
</p>


<br>

---

<br>

## 설계 포인트
#### 1. 빌드 로직을 단계별로 모듈화

복잡한 빌드 과정을 하나의 스크립트에 몰아넣지 않고, <br>
**사전 준비 → 이미지 빌드 → 결과 전파** 의 흐름으로 나누어 관리했습니다. <br>

이를 통해:
- 실패 지점을 빠르게 식별할 수 있고
- 단계별 수정 범위가 명확해지며
- 새로운 빌드 요구사항이 생겨도 특정 단계만 확장할 수 있습니다

<br>

#### 2. 사용자 코드 차이를 표준 빌드 흐름 안으로 흡수

FaaS 환경에서는 사용자가 제출하는 코드마다 실행 방식과 환경 구성이 다를 수 있습니다. <br>
이 차이를 수동 대응하지 않도록, 빌드 전에 다음 작업을 자동화했습니다.
- `package.json` 보정
- 함수 실행용 의존성 주입
- `.env` 파일 생성
- 런타임 진입점 래핑

즉, 사용자는 함수 코드만 제출하더라도, <br>
플랫폼은 이를 **실행 가능한 컨테이너 형태로 일관되게 변환**할 수 있습니다.

<br>

#### 3. 빌드 결과를 후속 시스템과 연결

빌드 성공 시 이미지를 푸시하는 데서 끝내지 않고,
**SQS 메시지 발행을 통해 다음 배포 단계가 이어지도록 구성**했습니다.

또한 실패 시에도 즉시 상태를 전파하도록 해,
단순 빌드 자동화가 아니라 **운영 가능한 파이프라인**이 되도록 만들었습니다.

<br>

---

<br>

## 디렉터리 구조

빌드 스크립트는 실행 단계와 책임에 따라 분리되어 있습니다.
```bash
.
├── buildspec.yml                    # AWS CodeBuild 빌드 명세
└── scripts
    ├── 01_prebuild/
    │   ├── install_pack.sh          # pack CLI 설치
    │   ├── resolve_env.sh           # 빌드 환경 변수 검증 및 설정
    │   ├── s3_sync_and_validate.sh  # S3 소스 다운로드 및 검증
    │   ├── patch_package_json.sh    # package.json 의존성/스크립트 주입
    │   └── process_env_files.sh     # 사용자 환경변수(.env) 생성 및 래핑
    ├── 02_build/
    │   └── build_image.sh           # pack build 실행 및 ECR 푸시
    ├── 03_postbuild/
    │   └── publish_sqs_message.sh   # SQS 성공 메시지 전송
    ├── lib/
    │   ├── create_env_file.py       # .env 파일 생성
    │   ├── create_env_wrapper.py    # dotenv 로드를 위한 엔트리포인트 래퍼 생성
    │   ├── parse_custom_env.py      # CUSTOM_ENV 파싱 유틸
    │   └── patch_package_json.py    # 의존성 추가 및 start 스크립트 주입
    └── common.sh                    # 공통 함수 (에러 핸들링, 환경변수 로드)
```

<br>

---

<br>

## 빌드 파이프라인
`buildspec.yml`은 4단계 라이프사이클로 구성되어 있습니다.

#### 1. Install
빌드에 필요한 공통 도구를 준비합니다.
- **install_pack.sh**
    - Cloud Native Buildpacks 사용을 위한 pack CLI 설치
- **캐시 활용**
    - `pack` 바이너리
    - Docker layer
    - 관련 볼륨 데이터 재사용

이 단계는 빌드 속도와 재실행 효율을 높이기 위한 기반입니다.

<br>

#### 2. Pre_build Phase
실제 빌드 전에, 사용자 코드를 플랫폼이 처리 가능한 형태로 정리합니다.
- **resolve_env.sh**
    - 필수 환경 변수 존재 여부 확인
    - 빌드에 필요한 값 로드 및 검증
- **s3_sync_and_validate.sh**
    - S3에서 사용자 소스 다운로드
    - 필요한 파일 존재 여부 및 무결성 확인
- **patch_package_json.sh**
    - `lib/patch_package_json.py` 호출
    - 함수 실행에 필요한 의존성과 start 스크립트 자동 주입
- **process_env_files.sh**
    - CUSTOM_ENV를 .env 파일로 변환
    - 런타임에서 이를 로드할 수 있도록 엔트리포인트 래핑
이 단계의 핵심은 사용자 코드의 편차를 사전에 흡수해 빌드 실패 가능성을 줄이는 것입니다.

<br>

#### 3. Build Phase

준비된 소스를 실제 컨테이너 이미지로 변환합니다.
- **build_image.sh**
    - `pack build` 실행
    - OCI 호환 이미지 생성
    - Amazon ECR 푸시
Cloud Native Buildpacks를 사용함으로써,
Dockerfile 없이도 일관된 방식으로 빌드 가능한 표준 컨테이너 경로를 제공합니다.

#### 4. Post_build Phase
준비된 소스를 실제 컨테이너 이미지로 변환합니다.
- **build_image.sh**
    - `pack build` 실행
    - OCI 호환 이미지 생성
    - Amazon ECR 푸시
Cloud Native Buildpacks를 사용함으로써,
Dockerfile 없이도 일관된 방식으로 빌드 가능한 표준 컨테이너 경로를 제공합니다.

<br>

---

<br>


## Python 유틸리티 스크립트 (`scripts/lib/`)

복잡한 파일 조작과 문자열 처리는 Python으로 분리해
쉘 스크립트의 역할을 단순화했습니다.

- **patch_package_json.py** <br>
    `package.json`에 필요한 의존성(`@google-cloud/functions-framework`, `dotenv`)과 실행 스크립트를 주입해 함수 실행 환경을 자동 보정합니다.
- **create_env_file.py** <br>
    CUSTOM_ENV 값을 .env 파일로 변환합니다.
- **create_env_wrapper.py** <br>
    `.env`를 로드한 뒤 기존 엔트리포인트를 실행할 수 있도록 래퍼 파일을 생성합니다.
- **parse_custom_env.py** <br>
    `CUSTOM_ENV`를 파싱해 `pack build --env` 방식으로 전달할 수 있도록 만든 대안 유틸입니다.
    현재 기본 빌드 경로에서는 사용하지 않습니다.

<br>

---

<br>


## 캐시 설정
빌드 시간을 줄이고 반복 실행 효율을 높이기 위해 로컬 캐시를 적용했습니다.
- **Source Cache** <br> git 소스 및 다운로드 파일 캐싱
- **Docker Layer Cache** <br> 이전 빌드 레이어 재사용
- **Custom Cache** <br> pack CLI 및 관련 데이터 캐싱
    - `/usr/local/bin/pack`
    - `/var/lib/docker/volumes`

<br>

---

<br>

## 기대 효과

이 저장소를 통해 빌드 파이프라인은 다음과 같은 특성을 갖습니다.
- 사용자 코드 차이를 흡수하는 **표준 빌드 구조**
- 단계별 책임 분리로 인한 **유지보수 용이성**
- 실패 지점 식별이 쉬운 **운영 친화적 구조**
- 빌드 결과를 후속 시스템과 연결하는 **확장 가능한 파이프라인**

즉, 이 리포지토리는 단순한 CodeBuild 설정이 아니라, <br>
FaaS 플랫폼에서 다양한 함수 코드를 **일관되게 빌드·배포 가능한 형태로 연결하는 빌드 오케스트레이션 레이어**입니다.
