# Isaac Lab Docker 완전 가이드

이 문서는 Isaac Lab의 `docker/container.py` 스크립트가 수행하는 **모든 작업**을 분석하고, 동등한 Docker 명령어를 제공합니다.

## 목차
1. [개요](#1-개요)
2. [주요 파일 설명](#2-주요-파일-설명)
3. [start 명령어 완전 분석](#3-start-명령어-완전-분석)
4. [이미지 빌드 (docker build)](#4-이미지-빌드-docker-build)
5. [컨테이너 생성 (docker run)](#5-컨테이너-생성-docker-run)
6. [컨테이너 진입 (docker exec)](#6-컨테이너-진입-docker-exec)
7. [기타 명령어 (stop, copy, config)](#7-기타-명령어-stop-copy-config)
8. [볼륨 상세 설명](#8-볼륨-상세-설명)
9. [X11 포워딩 상세](#9-x11-포워딩-상세)
10. [실전 예제](#10-실전-예제)

---

## 1. 개요

`docker/container.py` 스크립트는 다음 명령어를 지원합니다:

| 명령어 | 설명 |
|--------|------|
| `start [profile]` | 이미지 빌드(필요 시) + 컨테이너 생성 + 백그라운드 실행 |
| `enter [profile]` | 실행 중인 컨테이너에 bash 세션으로 진입 |
| `stop [profile]` | 컨테이너 중지 + **볼륨 삭제** + X11 정리 |
| `copy [profile]` | 컨테이너에서 호스트로 아티팩트 복사 |
| `config [profile]` | docker-compose 설정 출력/저장 |

**프로파일 종류:**
- `base` (기본값): Isaac Lab 기본 이미지
- `ros2`: ROS2 Humble이 추가된 이미지

---

## 2. 주요 파일 설명

```
docker/
├── container.py              # 메인 스크립트
├── docker-compose.yaml       # Docker Compose 설정
├── x11.yaml                  # X11 포워딩 추가 설정
├── Dockerfile.base           # base 프로파일 Dockerfile
├── Dockerfile.ros2           # ros2 프로파일 Dockerfile
├── .env.base                 # 기본 환경 변수
├── .env.ros2                 # ROS2 추가 환경 변수
├── .container.cfg            # 런타임 상태 저장 (X11 설정 등)
├── .isaac-lab-docker-history # bash 히스토리 공유 파일
└── utils/
    ├── container_interface.py  # Docker 작업 인터페이스
    ├── x11_utils.py            # X11 설정 유틸리티
    └── state_file.py           # 상태 파일 관리
```

---

## 3. start 명령어 완전 분석

`python3 docker/container.py start base` 실행 시 발생하는 일을 순서대로 설명합니다.

### 3.1 전체 흐름도

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. Docker 설치 확인                                              │
│    └─ shutil.which("docker") 체크                               │
├─────────────────────────────────────────────────────────────────┤
│ 2. ContainerInterface 초기화                                     │
│    ├─ 프로파일 설정 (base/ros2)                                  │
│    ├─ 환경 변수 로드 (.env.base)                                 │
│    └─ YAML 파일 목록 구성                                        │
├─────────────────────────────────────────────────────────────────┤
│ 3. X11 설정 확인 (x11_utils.x11_check)                          │
│    ├─ .container.cfg에서 X11_FORWARDING_ENABLED 확인            │
│    ├─ 첫 실행 시 사용자에게 Y/N 질문                             │
│    ├─ Y 선택 시:                                                 │
│    │   ├─ xauth 설치 확인                                        │
│    │   ├─ 임시 디렉토리 생성 (mktemp -d)                         │
│    │   ├─ .xauth 파일 생성 및 쿠키 설정                          │
│    │   └─ x11.yaml을 YAML 목록에 추가                            │
│    └─ 설정을 .container.cfg에 저장                               │
├─────────────────────────────────────────────────────────────────┤
│ 4. 히스토리 파일 생성                                            │
│    └─ .isaac-lab-docker-history 파일이 없으면 생성              │
├─────────────────────────────────────────────────────────────────┤
│ 5. Docker Compose 실행                                           │
│    └─ docker compose ... up --detach --build --remove-orphans   │
│       ├─ --build: 이미지가 없거나 변경되면 자동 빌드            │
│       ├─ --detach: 백그라운드 실행                               │
│       └─ --remove-orphans: 불필요한 컨테이너 제거               │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 실제 실행되는 Docker Compose 명령어

**X11 비활성화 시:**
```bash
docker compose \
  --file docker-compose.yaml \
  --profile base \
  --env-file .env.base \
  up --detach --build --remove-orphans
```

**X11 활성화 시:**
```bash
docker compose \
  --file docker-compose.yaml \
  --file x11.yaml \
  --profile base \
  --env-file .env.base \
  up --detach --build --remove-orphans
```

> **중요**: `--build` 플래그로 인해 이미지가 없으면 자동으로 빌드됩니다.

---

## 4. 이미지 빌드 (docker build)

### 4.1 사전 준비

docker-compose.yaml 없이 직접 `docker build`를 실행하려면 환경 변수를 먼저 설정해야 합니다.

```bash
# .env.base 파일의 내용을 환경 변수로 설정
export ISAACSIM_BASE_IMAGE=nvcr.io/nvidia/isaac-sim
export ISAACSIM_VERSION=5.0.0
export DOCKER_ISAACSIM_ROOT_PATH=/isaac-sim
export DOCKER_ISAACLAB_PATH=/workspace/isaaclab
export DOCKER_USER_HOME=/root

# Isaac Lab 프로젝트 루트 디렉토리로 이동
cd /home/user/IsaacLab-v221
```

### 4.2 동등한 docker build 명령어

```bash
docker build \
  --file docker/Dockerfile.base \
  --build-arg ISAACSIM_BASE_IMAGE_ARG=${ISAACSIM_BASE_IMAGE} \
  --build-arg ISAACSIM_VERSION_ARG=${ISAACSIM_VERSION} \
  --build-arg ISAACSIM_ROOT_PATH_ARG=${DOCKER_ISAACSIM_ROOT_PATH} \
  --build-arg ISAACLAB_PATH_ARG=${DOCKER_ISAACLAB_PATH} \
  --build-arg DOCKER_USER_HOME_ARG=${DOCKER_USER_HOME} \
  --tag isaac-lab-base:latest \
  .
```

### 4.3 복사 가능한 한 줄 명령어

```bash
cd /home/user/IsaacLab-v221 && docker build --file docker/Dockerfile.base --build-arg ISAACSIM_BASE_IMAGE_ARG=nvcr.io/nvidia/isaac-sim --build-arg ISAACSIM_VERSION_ARG=5.0.0 --build-arg ISAACSIM_ROOT_PATH_ARG=/isaac-sim --build-arg ISAACLAB_PATH_ARG=/workspace/isaaclab --build-arg DOCKER_USER_HOME_ARG=/root --tag isaac-lab-base:latest .
```

### 4.4 Dockerfile.base가 하는 일

1. NVIDIA Isaac Sim 베이스 이미지에서 시작 (`nvcr.io/nvidia/isaac-sim:5.0.0`)
2. 필수 패키지 설치 (build-essential, cmake, git, wget 등)
3. Isaac Lab 소스코드를 `/workspace/isaaclab`에 복사
4. Isaac Sim과 심볼릭 링크 생성
5. Isaac Lab 의존성 설치 (`isaaclab.sh --install`)
6. 편의용 alias 설정 (python, pip, isaaclab 등)

---

## 5. 컨테이너 생성 (docker run)

### 5.1 사전 준비 (공통)

```bash
# 환경 변수 설정
export DOCKER_ISAACSIM_ROOT_PATH=/isaac-sim
export DOCKER_ISAACLAB_PATH=/workspace/isaaclab
export DOCKER_USER_HOME=/root

# Isaac Lab 프로젝트 루트 디렉토리
ISAACLAB_DIR=/home/user/IsaacLab-v221

# 히스토리 파일 생성 (없으면)
touch ${ISAACLAB_DIR}/docker/.isaac-lab-docker-history
```

### 5.2 사전 준비 (X11 활성화 시 추가)

```bash
# xauth 설치 확인
which xauth || sudo apt install xauth

# 임시 xauth 파일 생성
TMP_DIR=$(mktemp -d)
TMP_XAUTH=$(mktemp --suffix=.xauth --tmpdir=${TMP_DIR})

# X11 인증 쿠키 추출 및 설정
xauth nlist ${DISPLAY} | sed 's/^....//' | xauth -f ${TMP_XAUTH} nmerge -

echo "TMP_DIR: ${TMP_DIR}"
echo "TMP_XAUTH: ${TMP_XAUTH}"
```

### 5.3 동등한 docker run 명령어 (전체 버전, X11 포함)

```bash
docker run \
  --detach \
  --interactive \
  --tty \
  --name isaac-lab-base \
  \
  # GPU 설정
  --gpus all \
  \
  # 네트워크 설정
  --network host \
  \
  # 환경 변수 - 기본
  -e ISAACSIM_PATH=${DOCKER_ISAACLAB_PATH}/_isaac_sim \
  -e OMNI_KIT_ALLOW_ROOT=1 \
  \
  # 환경 변수 - X11 (X11 활성화 시에만)
  -e DISPLAY=${DISPLAY} \
  -e TERM=${TERM} \
  -e QT_X11_NO_MITSHM=1 \
  -e XAUTHORITY=${TMP_XAUTH} \
  \
  # ========================================
  # 볼륨 마운트 - Named Volumes (Docker 관리)
  # ========================================
  # Isaac Sim 캐시 (읽기/쓰기)
  -v isaac-cache-kit:${DOCKER_ISAACSIM_ROOT_PATH}/kit/cache \
  -v isaac-cache-ov:${DOCKER_USER_HOME}/.cache/ov \
  -v isaac-cache-pip:${DOCKER_USER_HOME}/.cache/pip \
  -v isaac-cache-gl:${DOCKER_USER_HOME}/.cache/nvidia/GLCache \
  -v isaac-cache-compute:${DOCKER_USER_HOME}/.nv/ComputeCache \
  -v isaac-logs:${DOCKER_USER_HOME}/.nvidia-omniverse/logs \
  -v isaac-carb-logs:${DOCKER_ISAACSIM_ROOT_PATH}/kit/logs/Kit/Isaac-Sim \
  -v isaac-data:${DOCKER_USER_HOME}/.local/share/ov/data \
  -v isaac-docs:${DOCKER_USER_HOME}/Documents \
  \
  # Isaac Lab 아티팩트 (읽기/쓰기)
  -v isaac-lab-docs:${DOCKER_ISAACLAB_PATH}/docs/_build \
  -v isaac-lab-logs:${DOCKER_ISAACLAB_PATH}/logs \
  -v isaac-lab-data:${DOCKER_ISAACLAB_PATH}/data_storage \
  \
  # ========================================
  # 볼륨 마운트 - Bind Mounts (호스트 직접 연결)
  # ========================================
  # Isaac Lab 소스 - 실시간 동기화 (읽기/쓰기)
  -v ${ISAACLAB_DIR}/source:${DOCKER_ISAACLAB_PATH}/source \
  -v ${ISAACLAB_DIR}/scripts:${DOCKER_ISAACLAB_PATH}/scripts \
  -v ${ISAACLAB_DIR}/docs:${DOCKER_ISAACLAB_PATH}/docs \
  -v ${ISAACLAB_DIR}/tools:${DOCKER_ISAACLAB_PATH}/tools \
  \
  # Bash 히스토리 - 실시간 동기화 (읽기/쓰기)
  -v ${ISAACLAB_DIR}/docker/.isaac-lab-docker-history:${DOCKER_USER_HOME}/.bash_history \
  \
  # X11 볼륨 (X11 활성화 시에만)
  -v /tmp/.X11-unix:/tmp/.X11-unix \
  -v ${TMP_DIR}:${TMP_DIR} \
  -v /etc/localtime:/etc/localtime:ro \
  \
  # 작업 디렉토리
  --workdir ${DOCKER_ISAACLAB_PATH} \
  \
  # 엔트리포인트
  --entrypoint bash \
  \
  # 이미지
  isaac-lab-base:latest
```

### 5.4 복사 가능한 한 줄 명령어 (전체 버전, X11 포함)

```bash
# 사전 준비
ISAACLAB_DIR=/home/user/IsaacLab-v221 && touch ${ISAACLAB_DIR}/docker/.isaac-lab-docker-history && TMP_DIR=$(mktemp -d) && TMP_XAUTH=$(mktemp --suffix=.xauth --tmpdir=${TMP_DIR}) && xauth nlist ${DISPLAY} | sed 's/^....//' | xauth -f ${TMP_XAUTH} nmerge -

# 컨테이너 실행
docker run --detach --interactive --tty --name isaac-lab-base --gpus all --network host -e ISAACSIM_PATH=/workspace/isaaclab/_isaac_sim -e OMNI_KIT_ALLOW_ROOT=1 -e DISPLAY=${DISPLAY} -e TERM=${TERM} -e QT_X11_NO_MITSHM=1 -e XAUTHORITY=${TMP_XAUTH} -v isaac-cache-kit:/isaac-sim/kit/cache -v isaac-cache-ov:/root/.cache/ov -v isaac-cache-pip:/root/.cache/pip -v isaac-cache-gl:/root/.cache/nvidia/GLCache -v isaac-cache-compute:/root/.nv/ComputeCache -v isaac-logs:/root/.nvidia-omniverse/logs -v isaac-carb-logs:/isaac-sim/kit/logs/Kit/Isaac-Sim -v isaac-data:/root/.local/share/ov/data -v isaac-docs:/root/Documents -v isaac-lab-docs:/workspace/isaaclab/docs/_build -v isaac-lab-logs:/workspace/isaaclab/logs -v isaac-lab-data:/workspace/isaaclab/data_storage -v ${ISAACLAB_DIR}/source:/workspace/isaaclab/source -v ${ISAACLAB_DIR}/scripts:/workspace/isaaclab/scripts -v ${ISAACLAB_DIR}/docs:/workspace/isaaclab/docs -v ${ISAACLAB_DIR}/tools:/workspace/isaaclab/tools -v ${ISAACLAB_DIR}/docker/.isaac-lab-docker-history:/root/.bash_history -v /tmp/.X11-unix:/tmp/.X11-unix -v ${TMP_DIR}:${TMP_DIR} -v /etc/localtime:/etc/localtime:ro --workdir /workspace/isaaclab --entrypoint bash isaac-lab-base:latest
```

### 5.5 최소 버전 (X11 제외, Headless 모드)

```bash
ISAACLAB_DIR=/home/user/IsaacLab-v221 && touch ${ISAACLAB_DIR}/docker/.isaac-lab-docker-history && docker run --detach --interactive --tty --name isaac-lab-base --gpus all --network host -e ISAACSIM_PATH=/workspace/isaaclab/_isaac_sim -e OMNI_KIT_ALLOW_ROOT=1 -v isaac-cache-kit:/isaac-sim/kit/cache -v isaac-cache-ov:/root/.cache/ov -v isaac-cache-pip:/root/.cache/pip -v isaac-cache-gl:/root/.cache/nvidia/GLCache -v isaac-cache-compute:/root/.nv/ComputeCache -v isaac-logs:/root/.nvidia-omniverse/logs -v isaac-carb-logs:/isaac-sim/kit/logs/Kit/Isaac-Sim -v isaac-data:/root/.local/share/ov/data -v isaac-docs:/root/Documents -v isaac-lab-docs:/workspace/isaaclab/docs/_build -v isaac-lab-logs:/workspace/isaaclab/logs -v isaac-lab-data:/workspace/isaaclab/data_storage -v ${ISAACLAB_DIR}/source:/workspace/isaaclab/source -v ${ISAACLAB_DIR}/scripts:/workspace/isaaclab/scripts -v ${ISAACLAB_DIR}/docs:/workspace/isaaclab/docs -v ${ISAACLAB_DIR}/tools:/workspace/isaaclab/tools -v ${ISAACLAB_DIR}/docker/.isaac-lab-docker-history:/root/.bash_history --workdir /workspace/isaaclab --entrypoint bash isaac-lab-base:latest
```

---

## 6. 컨테이너 진입 (docker exec)

### 6.1 enter 명령어가 하는 일

`python3 docker/container.py enter base` 실행 시:

1. **X11 쿠키 갱신** (`x11_utils.x11_refresh`)
   - SSH 재접속 등으로 `DISPLAY` 값이 변경되었을 수 있음
   - 기존 `.xauth` 파일 삭제 후 새로 생성
2. **컨테이너 진입**
   - 현재 `DISPLAY` 값을 동적으로 전달

### 6.2 동등한 docker exec 명령어

**X11 활성화 시:**
```bash
# X11 쿠키 갱신 (선택적이지만 권장)
TMP_XAUTH=$(cat /home/user/IsaacLab-v221/docker/.container.cfg | grep __isaaclab_tmp_xauth | cut -d= -f2)
if [ -n "${TMP_XAUTH}" ] && [ -f "${TMP_XAUTH}" ]; then
  rm -f ${TMP_XAUTH}
  xauth nlist ${DISPLAY} | sed 's/^....//' | xauth -f ${TMP_XAUTH} nmerge -
fi

# 컨테이너 진입
docker exec \
  --interactive \
  --tty \
  -e DISPLAY=${DISPLAY} \
  isaac-lab-base \
  bash
```

**X11 비활성화 시 (간단):**
```bash
docker exec --interactive --tty isaac-lab-base bash
```

### 6.3 복사 가능한 한 줄 명령어

```bash
docker exec --interactive --tty -e DISPLAY=${DISPLAY} isaac-lab-base bash
```

---

## 7. 기타 명령어 (stop, copy, config)

### 7.1 stop 명령어

`python3 docker/container.py stop base` 실행 시:

**실행되는 Docker Compose 명령어:**
```bash
docker compose \
  --file docker-compose.yaml \
  [--file x11.yaml] \
  --profile base \
  --env-file .env.base \
  down --volumes
```

**정리되는 것들:**

| 항목 | 삭제 여부 | 설명 |
|------|-----------|------|
| 컨테이너 | **삭제됨** | `isaac-lab-base` 컨테이너 제거 |
| Named Volumes | **삭제됨** | `--volumes` 플래그로 인해 모든 named volume 삭제 |
| Bind Mounts | **유지됨** | 호스트의 실제 파일이므로 영향 없음 |
| X11 임시 파일 | **삭제됨** | `.xauth` 파일 및 임시 디렉토리 제거 |
| 네트워크 | **삭제됨** | Docker Compose가 생성한 네트워크 제거 |
| 이미지 | **유지됨** | 이미지는 삭제되지 않음 |

**동등한 docker 명령어:**
```bash
# 컨테이너 중지 및 제거
docker stop isaac-lab-base
docker rm isaac-lab-base

# Named volumes 삭제 (선택적 - stop 시 자동 삭제됨)
docker volume rm isaac-cache-kit isaac-cache-ov isaac-cache-pip isaac-cache-gl \
  isaac-cache-compute isaac-logs isaac-carb-logs isaac-data isaac-docs \
  isaac-lab-docs isaac-lab-logs isaac-lab-data

# X11 임시 파일 정리 (X11 활성화 시)
TMP_XAUTH=$(cat .container.cfg 2>/dev/null | grep __isaaclab_tmp_xauth | cut -d= -f2)
[ -n "${TMP_XAUTH}" ] && rm -f "${TMP_XAUTH}" && rmdir "$(dirname ${TMP_XAUTH})" 2>/dev/null
```

> **주의**: `stop` 명령은 **Named Volumes를 삭제**합니다! 캐시와 로그가 모두 사라집니다.
> 볼륨을 유지하려면 `docker stop isaac-lab-base`만 실행하세요.

### 7.2 copy 명령어

`python3 docker/container.py copy base` 실행 시:

컨테이너에서 호스트로 아티팩트를 복사합니다.

```bash
# 복사되는 경로
/workspace/isaaclab/logs       → docker/artifacts/logs
/workspace/isaaclab/docs/_build → docker/artifacts/docs
/workspace/isaaclab/data_storage → docker/artifacts/data_storage
```

**동등한 docker 명령어:**
```bash
mkdir -p docker/artifacts
docker cp isaac-lab-base:/workspace/isaaclab/logs docker/artifacts/logs
docker cp isaac-lab-base:/workspace/isaaclab/docs/_build docker/artifacts/docs
docker cp isaac-lab-base:/workspace/isaaclab/data_storage docker/artifacts/data_storage
```

### 7.3 config 명령어

`python3 docker/container.py config base` 실행 시:

Docker Compose 설정을 병합하여 출력합니다.

```bash
docker compose \
  --file docker-compose.yaml \
  --profile base \
  --env-file .env.base \
  config
```

YAML 파일로 저장:
```bash
docker compose \
  --file docker-compose.yaml \
  --profile base \
  --env-file .env.base \
  config --output merged-config.yaml
```

---

## 8. 볼륨 상세 설명

### 8.1 볼륨 타입 비교

| 타입 | 설명 | 실시간 동기화 | 호스트 위치 |
|------|------|---------------|-------------|
| **Named Volume** | Docker가 관리하는 볼륨 | 아니오 (컨테이너 내부 전용) | `/var/lib/docker/volumes/<name>/_data` |
| **Bind Mount** | 호스트 디렉토리 직접 연결 | **예** (양방향 동기화) | 지정한 호스트 경로 |

### 8.2 Named Volumes 상세

Named Volume은 Docker가 관리하는 볼륨으로, **컨테이너 생성 시 자동 생성**됩니다.

```bash
# Named Volume 위치 확인
docker volume inspect isaac-cache-kit
# 출력: "Mountpoint": "/var/lib/docker/volumes/isaac-cache-kit/_data"

# 모든 Isaac 관련 볼륨 확인
docker volume ls | grep isaac
```

| 볼륨 이름 | 컨테이너 경로 | 용도 | 읽기/쓰기 |
|-----------|---------------|------|-----------|
| `isaac-cache-kit` | `/isaac-sim/kit/cache` | Kit 캐시 | 읽기/쓰기 |
| `isaac-cache-ov` | `/root/.cache/ov` | Omniverse 캐시 | 읽기/쓰기 |
| `isaac-cache-pip` | `/root/.cache/pip` | pip 캐시 | 읽기/쓰기 |
| `isaac-cache-gl` | `/root/.cache/nvidia/GLCache` | OpenGL 캐시 | 읽기/쓰기 |
| `isaac-cache-compute` | `/root/.nv/ComputeCache` | CUDA 컴파일 캐시 | 읽기/쓰기 |
| `isaac-logs` | `/root/.nvidia-omniverse/logs` | Omniverse 로그 | 읽기/쓰기 |
| `isaac-carb-logs` | `/isaac-sim/kit/logs/Kit/Isaac-Sim` | Carbonite 로그 | 읽기/쓰기 |
| `isaac-data` | `/root/.local/share/ov/data` | Omniverse 데이터 | 읽기/쓰기 |
| `isaac-docs` | `/root/Documents` | 문서 | 읽기/쓰기 |
| `isaac-lab-docs` | `/workspace/isaaclab/docs/_build` | 빌드된 문서 | 읽기/쓰기 |
| `isaac-lab-logs` | `/workspace/isaaclab/logs` | 학습 로그 | 읽기/쓰기 |
| `isaac-lab-data` | `/workspace/isaaclab/data_storage` | 데이터 저장소 | 읽기/쓰기 |

> **Q: `isaac-docs:/root/Documents`에서 `isaac-docs`는 호스트의 어디인가요?**
>
> A: `/var/lib/docker/volumes/isaac-docs/_data`에 저장됩니다. Docker가 관리하는 볼륨이므로
> 호스트의 특정 디렉토리를 지정한 것이 아닙니다. `docker volume inspect isaac-docs`로 확인 가능합니다.

### 8.3 Bind Mounts 상세

Bind Mount는 호스트 디렉토리를 **직접 연결**합니다. **양방향 실시간 동기화**가 됩니다.

| 호스트 경로 | 컨테이너 경로 | 용도 | 읽기/쓰기 |
|-------------|---------------|------|-----------|
| `${ISAACLAB_DIR}/source` | `/workspace/isaaclab/source` | 소스코드 | 읽기/쓰기 |
| `${ISAACLAB_DIR}/scripts` | `/workspace/isaaclab/scripts` | 스크립트 | 읽기/쓰기 |
| `${ISAACLAB_DIR}/docs` | `/workspace/isaaclab/docs` | 문서 소스 | 읽기/쓰기 |
| `${ISAACLAB_DIR}/tools` | `/workspace/isaaclab/tools` | 도구 | 읽기/쓰기 |
| `${ISAACLAB_DIR}/docker/.isaac-lab-docker-history` | `/root/.bash_history` | Bash 히스토리 | 읽기/쓰기 |
| `/tmp/.X11-unix` | `/tmp/.X11-unix` | X11 소켓 | 읽기/쓰기 |
| `${TMP_DIR}` | `${TMP_DIR}` | xauth 디렉토리 | 읽기/쓰기 |
| `/etc/localtime` | `/etc/localtime` | 시간대 | **읽기 전용** |

> **Q: Isaac Lab 소스 볼륨은 실시간 동기화인가요?**
>
> A: **예, 실시간 동기화입니다.** Bind Mount이므로:
> - 호스트에서 파일 수정 → 컨테이너에서 즉시 반영
> - 컨테이너에서 파일 수정 → 호스트에서 즉시 반영
>
> **단, 컨테이너 생성 후 `ISAACLAB_DIR` 변수를 변경해도 기존 컨테이너에는 영향이 없습니다.**
> 마운트는 컨테이너 생성 시점에 결정되며, 컨테이너를 재생성해야 새 경로가 적용됩니다.

### 8.4 볼륨 관련 주의사항

1. **Named Volume은 컨테이너보다 오래 유지됨**
   - `docker stop`으로 컨테이너를 중지해도 볼륨은 유지
   - `docker rm`으로 컨테이너를 삭제해도 볼륨은 유지
   - `docker compose down --volumes` 또는 `docker volume rm`으로만 삭제

2. **container.py stop은 볼륨을 삭제함**
   - `--volumes` 플래그가 사용되므로 Named Volume이 모두 삭제됨
   - 캐시를 유지하려면 `docker stop isaac-lab-base`만 실행

3. **Bind Mount 경로 변경 시**
   - 기존 컨테이너에는 영향 없음
   - 새 경로를 적용하려면 컨테이너 재생성 필요

---

## 9. X11 포워딩 상세

### 9.1 작동 원리

X11 포워딩은 **X11 소켓 공유**와 **MIT-MAGIC-COOKIE 인증**을 조합하여 구현됩니다.

```
┌─────────────────────────────────────────────────────────────────┐
│ 호스트                                                          │
│                                                                 │
│  X Server ◄──── /tmp/.X11-unix/X0 ────┐                        │
│     ▲                                  │                        │
│     │ 인증 (MIT-MAGIC-COOKIE)         │ Unix Socket            │
│     │                                  │                        │
│  ~/.Xauthority                         │                        │
│     │                                  │                        │
│     ▼                                  │                        │
│  xauth nlist $DISPLAY                  │                        │
│     │                                  │                        │
│     ▼                                  ▼                        │
│  /tmp/xxx.xauth  ═══════════════► 컨테이너                      │
│  (임시 쿠키 파일)                      │                        │
│                                        │                        │
│                     ┌──────────────────┘                        │
│                     ▼                                           │
│              ┌─────────────┐                                    │
│              │  GUI 앱     │                                    │
│              │  (Isaac Sim)│                                    │
│              └─────────────┘                                    │
└─────────────────────────────────────────────────────────────────┘
```

### 9.2 핵심 구성 요소

| 구성 요소 | 설명 |
|-----------|------|
| `DISPLAY=$DISPLAY` | 사용할 X 디스플레이 (예: `:0`, `localhost:10.0`) |
| `/tmp/.X11-unix` 마운트 | X11 Unix 소켓 공유 |
| `XAUTHORITY` | 인증 쿠키 파일 경로 |
| `.xauth` 파일 마운트 | 인증 쿠키 파일 공유 |
| `QT_X11_NO_MITSHM=1` | Qt 앱의 공유 메모리 비활성화 (컨테이너 호환성) |

### 9.3 수동으로 X11 설정하기

```bash
# 1. xauth 설치 확인
which xauth || sudo apt install xauth

# 2. 임시 xauth 파일 생성
TMP_DIR=$(mktemp -d)
TMP_XAUTH=$(mktemp --suffix=.xauth --tmpdir=${TMP_DIR})

# 3. 현재 X11 쿠키를 범용 주소화하여 복사
# 'ffff'를 제거하면 모든 호스트에서 사용 가능한 쿠키가 됨
xauth nlist ${DISPLAY} | sed 's/^....//' | xauth -f ${TMP_XAUTH} nmerge -

# 4. 확인
echo "DISPLAY: ${DISPLAY}"
echo "TMP_XAUTH: ${TMP_XAUTH}"
xauth -f ${TMP_XAUTH} list
```

### 9.4 SSH를 통한 X11 포워딩

SSH로 원격 서버에 접속할 때 X11 포워딩을 사용하려면:

```bash
# 클라이언트에서 SSH 접속 (-X 또는 -Y 옵션)
ssh -X user@remote-server

# 서버에서 DISPLAY 확인 (예: localhost:10.0)
echo $DISPLAY

# 이 DISPLAY 값이 컨테이너에 전달됨
```

---

## 10. 실전 예제

### 10.1 최소 Headless 컨테이너 (X11 없음)

GUI가 필요 없는 학습/추론 환경:

```bash
ISAACLAB_DIR=/home/user/IsaacLab-v221

# 히스토리 파일 생성
touch ${ISAACLAB_DIR}/docker/.isaac-lab-docker-history

# 컨테이너 실행
docker run -dit --name isaac-lab-headless --gpus all --network host \
  -e ISAACSIM_PATH=/workspace/isaaclab/_isaac_sim \
  -e OMNI_KIT_ALLOW_ROOT=1 \
  -v isaac-cache-kit:/isaac-sim/kit/cache \
  -v isaac-cache-ov:/root/.cache/ov \
  -v isaac-cache-pip:/root/.cache/pip \
  -v ${ISAACLAB_DIR}/source:/workspace/isaaclab/source \
  -v ${ISAACLAB_DIR}/scripts:/workspace/isaaclab/scripts \
  --workdir /workspace/isaaclab \
  --entrypoint bash \
  isaac-lab-base:latest

# 진입
docker exec -it isaac-lab-headless bash
```

### 10.2 커스텀 볼륨 추가

추가 데이터 디렉토리를 마운트하려면:

```bash
# 예: 호스트의 /data/datasets를 컨테이너의 /workspace/datasets에 마운트
docker run -dit --name isaac-lab-custom --gpus all --network host \
  ... (기존 옵션들) ... \
  -v /data/datasets:/workspace/datasets \
  -v /data/checkpoints:/workspace/checkpoints:ro \  # 읽기 전용
  isaac-lab-base:latest
```

### 10.3 다른 Isaac Sim 버전 사용

```bash
# .env.base 수정 또는 빌드 시 직접 지정
docker build \
  --file docker/Dockerfile.base \
  --build-arg ISAACSIM_BASE_IMAGE_ARG=nvcr.io/nvidia/isaac-sim \
  --build-arg ISAACSIM_VERSION_ARG=4.2.0 \  # 다른 버전
  ... \
  --tag isaac-lab-base:4.2.0 \
  .
```

### 10.4 볼륨 없이 일회용 컨테이너

```bash
docker run -it --rm --gpus all --network host \
  -e ISAACSIM_PATH=/workspace/isaaclab/_isaac_sim \
  -e OMNI_KIT_ALLOW_ROOT=1 \
  --workdir /workspace/isaaclab \
  isaac-lab-base:latest \
  bash
```

> `--rm` 옵션으로 컨테이너 종료 시 자동 삭제됩니다.

### 10.5 컨테이너 상태 확인 명령어

```bash
# 실행 중인 컨테이너 확인
docker ps | grep isaac

# 모든 컨테이너 확인 (중지된 것 포함)
docker ps -a | grep isaac

# 볼륨 확인
docker volume ls | grep isaac

# 이미지 확인
docker images | grep isaac

# 컨테이너 로그 확인
docker logs isaac-lab-base

# 컨테이너 리소스 사용량
docker stats isaac-lab-base
```

---

## 부록: 빠른 참조

### A. 환경 변수 요약

| 변수 | 기본값 | 설명 |
|------|--------|------|
| `ISAACSIM_BASE_IMAGE` | `nvcr.io/nvidia/isaac-sim` | 베이스 이미지 |
| `ISAACSIM_VERSION` | `5.0.0` | Isaac Sim 버전 |
| `DOCKER_ISAACSIM_ROOT_PATH` | `/isaac-sim` | 컨테이너 내 Isaac Sim 경로 |
| `DOCKER_ISAACLAB_PATH` | `/workspace/isaaclab` | 컨테이너 내 Isaac Lab 경로 |
| `DOCKER_USER_HOME` | `/root` | 컨테이너 내 홈 디렉토리 |

### B. 주요 파일 경로

| 호스트 | 컨테이너 |
|--------|----------|
| `/home/user/IsaacLab-v221` | `/workspace/isaaclab` |
| `docker/.container.cfg` | (호스트 전용) |
| `docker/.isaac-lab-docker-history` | `/root/.bash_history` |

### C. 문제 해결

**Q: X11 오류 발생 시**
```bash
# DISPLAY 확인
echo $DISPLAY

# xauth 목록 확인
xauth list

# xhost로 모든 연결 허용 (보안 취약, 테스트용)
xhost +local:
```

**Q: GPU 인식 안 됨**
```bash
# NVIDIA 드라이버 확인
nvidia-smi

# Docker에서 GPU 접근 테스트
docker run --rm --gpus all nvidia/cuda:11.0-base nvidia-smi
```

**Q: 볼륨 권한 문제**
```bash
# Named volume 내용 확인 (root 권한 필요할 수 있음)
sudo ls -la /var/lib/docker/volumes/isaac-cache-kit/_data
```
