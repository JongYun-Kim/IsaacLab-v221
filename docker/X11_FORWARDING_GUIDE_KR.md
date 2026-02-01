# Docker X11 포워딩 가이드 (Isaac Lab)

이 문서는 Isaac Lab의 `docker/container.py` 스크립트에서 X11 포워딩이 어떻게 작동하는지 설명합니다.

## 목차
1. [X11 포워딩 작동 원리](#1-x11-포워딩-작동-원리)
2. [최소 예제](#2-최소-예제)
3. [전체 Docker 명령어](#3-전체-docker-명령어)

---

## 1. X11 포워딩 작동 원리

Isaac Lab Docker의 X11 포워딩은 **X11 소켓 공유**와 **MIT-MAGIC-COOKIE 인증**을 조합하여 구현됩니다.

### 1.1 X11 인증 쿠키 생성 (컨테이너 시작 시)

`python3 docker/container.py start base` 실행 시:

1. **호스트에 `xauth` 설치 여부 확인**
2. **임시 디렉토리 생성**: `mktemp -d`
3. **임시 `.xauth` 파일 생성** (예: `/tmp/tmp.XXXX/YYYY.xauth`)
4. **현재 X11 인증 쿠키 추출**:
   ```bash
   xauth nlist $DISPLAY
   ```
5. **쿠키를 "범용 주소"로 변환**: `ffff` 프리픽스 제거 (모든 호스트에서 사용 가능)
6. **임시 `.xauth` 파일에 쿠키 병합**:
   ```bash
   xauth -f /tmp/tmp.XXXX/YYYY.xauth nmerge -
   ```

### 1.2 Docker Compose 설정 (x11.yaml)

**환경 변수:**
| 변수 | 값 | 용도 |
|------|-----|------|
| `DISPLAY` | 호스트에서 상속 (예: `:0`) | GUI 앱에 사용할 X 디스플레이 지정 |
| `TERM` | 호스트에서 상속 | 터미널 타입 |
| `QT_X11_NO_MITSHM` | `1` | Qt 앱의 공유 메모리 비활성화 (필수) |
| `XAUTHORITY` | `/tmp/tmp.XXXX/YYYY.xauth` | 인증 쿠키 파일 경로 |

**볼륨 마운트:**
| 호스트 경로 | 컨테이너 경로 | 용도 |
|-------------|---------------|------|
| `/tmp/.X11-unix` | `/tmp/.X11-unix` | **X11 Unix 소켓** - GUI 앱과 X 서버 통신 채널 |
| 임시 xauth 디렉토리 | 동일 경로 | 컨테이너 내부에서 `.xauth` 파일 접근 |
| `/etc/localtime` | `/etc/localtime` (읽기전용) | 시간대 동기화 |

### 1.3 컨테이너 진입 시

`python3 docker/container.py enter base` 실행 시:

1. **X11 쿠키 갱신**: SSH 재접속 등으로 `DISPLAY` 값이 변경되었을 수 있으므로 쿠키 재생성
2. **현재 `DISPLAY` 값으로 컨테이너 진입**:
   ```bash
   docker exec --interactive --tty -e DISPLAY=$DISPLAY isaac-lab-base bash
   ```

### 1.4 GUI 앱 실행 흐름

컨테이너 내부에서:
1. 앱이 `DISPLAY` 환경 변수 읽음 (예: `:0`)
2. `/tmp/.X11-unix/X0` Unix 소켓에 연결
3. `XAUTHORITY`에서 인증 쿠키 읽음
4. X 서버에 쿠키 전송하여 인증
5. X 서버가 쿠키 검증 후 렌더링 허용

---

## 2. 최소 예제

다른 Docker 컨테이너에 X11 포워딩을 적용하기 위한 최소 설정:

```bash
# 1. 임시 xauth 파일 생성
XAUTH_FILE=$(mktemp --suffix=.xauth)

# 2. X11 쿠키 추출 및 수정 (범용 주소화)
xauth nlist $DISPLAY | sed 's/^..../ffff/' | xauth -f $XAUTH_FILE nmerge -

# 3. X11 포워딩이 활성화된 컨테이너 실행
docker run -it --rm \
  -e DISPLAY=$DISPLAY \
  -e XAUTHORITY=$XAUTH_FILE \
  -e QT_X11_NO_MITSHM=1 \
  -v /tmp/.X11-unix:/tmp/.X11-unix \
  -v $XAUTH_FILE:$XAUTH_FILE \
  ubuntu:22.04 bash

# 4. 컨테이너 내부에서 테스트:
#    apt update && apt install -y x11-apps && xeyes
```

**대안 (더 간단하지만 보안 취약):**
```bash
# 모든 로컬 연결을 X 서버에 허용 (보안 위험!)
xhost +local:

docker run -it --rm \
  -e DISPLAY=$DISPLAY \
  -v /tmp/.X11-unix:/tmp/.X11-unix \
  --network host \
  ubuntu:22.04 bash
```

---

## 3. 전체 Docker 명령어

### 3.1 `python3 docker/container.py start base` 동등 명령어

**사전 준비:**
```bash
# 임시 xauth 파일 생성 (x11_utils.py 동작 모방)
TMP_DIR=$(mktemp -d)
TMP_XAUTH=$(mktemp --suffix=.xauth --tmpdir=$TMP_DIR)
xauth nlist $DISPLAY | sed 's/^....//' | xauth -f $TMP_XAUTH nmerge -

# 히스토리 파일 생성
touch $(pwd)/docker/.isaac-lab-docker-history
```

**전체 `docker run` 명령어:**
```bash
docker run \
  --detach \
  --interactive \
  --tty \
  --name isaac-lab-base \
  --network host \
  --gpus all \
  \
  # 환경 변수
  -e ISAACSIM_PATH=/workspace/isaaclab/_isaac_sim \
  -e OMNI_KIT_ALLOW_ROOT=1 \
  -e DISPLAY=$DISPLAY \
  -e TERM=$TERM \
  -e QT_X11_NO_MITSHM=1 \
  -e XAUTHORITY=$TMP_XAUTH \
  \
  # X11 볼륨
  -v /tmp/.X11-unix:/tmp/.X11-unix \
  -v $TMP_DIR:$TMP_DIR \
  -v /etc/localtime:/etc/localtime:ro \
  \
  # Isaac Sim 캐시 볼륨
  -v isaac-cache-kit:/isaac-sim/kit/cache \
  -v isaac-cache-ov:/root/.cache/ov \
  -v isaac-cache-pip:/root/.cache/pip \
  -v isaac-cache-gl:/root/.cache/nvidia/GLCache \
  -v isaac-cache-compute:/root/.nv/ComputeCache \
  -v isaac-logs:/root/.nvidia-omniverse/logs \
  -v isaac-carb-logs:/isaac-sim/kit/logs/Kit/Isaac-Sim \
  -v isaac-data:/root/.local/share/ov/data \
  -v isaac-docs:/root/Documents \
  \
  # Isaac Lab 소스 바인드 마운트
  -v $(pwd)/source:/workspace/isaaclab/source \
  -v $(pwd)/scripts:/workspace/isaaclab/scripts \
  -v $(pwd)/docs:/workspace/isaaclab/docs \
  -v $(pwd)/tools:/workspace/isaaclab/tools \
  \
  # Isaac Lab 아티팩트 볼륨
  -v isaac-lab-docs:/workspace/isaaclab/docs/_build \
  -v isaac-lab-logs:/workspace/isaaclab/logs \
  -v isaac-lab-data:/workspace/isaaclab/data_storage \
  \
  # Bash 히스토리 유지
  -v $(pwd)/docker/.isaac-lab-docker-history:/root/.bash_history \
  \
  # 작업 디렉토리
  -w /workspace/isaaclab \
  \
  # 엔트리포인트
  --entrypoint bash \
  \
  # 이미지 (이미 빌드되어 있다고 가정)
  isaac-lab-base:latest
```

**복사 가능한 한 줄 명령어:**
```bash
TMP_DIR=$(mktemp -d) && TMP_XAUTH=$(mktemp --suffix=.xauth --tmpdir=$TMP_DIR) && xauth nlist $DISPLAY | sed 's/^....//' | xauth -f $TMP_XAUTH nmerge - && touch $(pwd)/docker/.isaac-lab-docker-history && docker run --detach --interactive --tty --name isaac-lab-base --network host --gpus all -e ISAACSIM_PATH=/workspace/isaaclab/_isaac_sim -e OMNI_KIT_ALLOW_ROOT=1 -e DISPLAY=$DISPLAY -e TERM=$TERM -e QT_X11_NO_MITSHM=1 -e XAUTHORITY=$TMP_XAUTH -v /tmp/.X11-unix:/tmp/.X11-unix -v $TMP_DIR:$TMP_DIR -v /etc/localtime:/etc/localtime:ro -v isaac-cache-kit:/isaac-sim/kit/cache -v isaac-cache-ov:/root/.cache/ov -v isaac-cache-pip:/root/.cache/pip -v isaac-cache-gl:/root/.cache/nvidia/GLCache -v isaac-cache-compute:/root/.nv/ComputeCache -v isaac-logs:/root/.nvidia-omniverse/logs -v isaac-carb-logs:/isaac-sim/kit/logs/Kit/Isaac-Sim -v isaac-data:/root/.local/share/ov/data -v isaac-docs:/root/Documents -v $(pwd)/source:/workspace/isaaclab/source -v $(pwd)/scripts:/workspace/isaaclab/scripts -v $(pwd)/docs:/workspace/isaaclab/docs -v $(pwd)/tools:/workspace/isaaclab/tools -v isaac-lab-docs:/workspace/isaaclab/docs/_build -v isaac-lab-logs:/workspace/isaaclab/logs -v isaac-lab-data:/workspace/isaaclab/data_storage -v $(pwd)/docker/.isaac-lab-docker-history:/root/.bash_history -w /workspace/isaaclab --entrypoint bash isaac-lab-base:latest
```

### 3.2 `python3 docker/container.py enter base` 동등 명령어

```bash
docker exec \
  --interactive \
  --tty \
  -e DISPLAY=$DISPLAY \
  isaac-lab-base \
  bash
```

**중요**: `DISPLAY` 변수는 컨테이너 진입 시 **동적으로 전달**해야 합니다. 컨테이너 시작 이후 SSH 재접속 등으로 `DISPLAY` 값이 변경되었을 수 있기 때문입니다.

---

## 핵심 X11 컴포넌트 요약

| 컴포넌트 | 용도 | 필수 여부 |
|----------|------|-----------|
| `-v /tmp/.X11-unix:/tmp/.X11-unix` | X11 Unix 소켓 공유 | **필수** |
| `-e DISPLAY=$DISPLAY` | 사용할 디스플레이 지정 | **필수** |
| `-e XAUTHORITY=...` | 인증 쿠키 파일 경로 | 권장 (보안 인증용) |
| `-v $XAUTH_FILE:$XAUTH_FILE` | 컨테이너 내 쿠키 접근 | 권장 (보안 인증용) |
| `-e QT_X11_NO_MITSHM=1` | Qt 앱 컨테이너 호환성 | 권장 |
| `--network host` | 네트워킹 단순화 | 선택 (일부 앱에 도움) |

---

## 참고 파일

- `docker/container.py`: 메인 컨테이너 관리 스크립트
- `docker/utils/x11_utils.py`: X11 인증 처리 유틸리티
- `docker/utils/container_interface.py`: Docker 컨테이너 인터페이스
- `docker/x11.yaml`: X11 Docker Compose 설정
- `docker/docker-compose.yaml`: 기본 Docker Compose 설정
