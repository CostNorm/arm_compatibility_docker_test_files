# arm_compatibility_docker_test_files

`DockerAnalyzer`의 다양한 기능을 테스트하기에 적합한 여러 시나리오를 포함하는 Dockerfile 예시들. 이 파일들을 개인 GitHub 리포지토리에 올리고 분석기를 실행하여 결과를 확인해 보세요.

---

**1. `Dockerfile.good` (높은 ARM 마이그레이션 잠재력)**

- **의도:** ARM64를 네이티브하게 지원하는 베이스 이미지를 사용하고, `--platform=linux/amd64` 플래그가 있더라도 다른 심각한 문제가 없어 마이그레이션이 유력한 케이스를 테스트합니다. 멀티라인 명령어와 `TARGETARCH` 사용도 포함합니다.
- **예상 결과:**
  - 베이스 이미지(`python:3.9-slim-buster`)는 ARM64 지원 (`compatible: True`).
  - `--platform` 플래그 감지 및 제거 권장.
  - `TARGETARCH` 사용은 긍정적 신호로 간주될 수 있음 (또는 최소한 문제 없음).
  - 다른 심각한 x86 의존성 없음.
  - **Overall Potential: High**

```dockerfile
# Dockerfile.good: ARM 마이그레이션 가능성이 높은 시나리오
# 베이스 이미지는 multi-arch를 지원하지만, --platform 플래그가 있음
FROM --platform=linux/amd64 python:3.9-slim-buster AS builder

WORKDIR /app

# 표준 패키지 설치 (ARM에서도 문제 없음)
# 멀티라인 RUN 명령어 테스트
RUN apt-get update && \
    apt-get install -y --no-install-recommends gcc build-essential && \
    rm -rf /var/lib/apt/lists/*

# pip 설치 (ARM 호환)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# TARGETARCH 사용 예시 (좋은 패턴)
ARG TARGETARCH
RUN echo "Building for architecture: ${TARGETARCH:-amd64}"

COPY . .

# 간단한 두 번째 스테이지 (multi-arch 베이스)
FROM python:3.9-slim-buster

WORKDIR /app

COPY --from=builder /app /app

# 주석 처리된 라인
# RUN dpkg --add-architecture amd64

# 빈 줄

CMD ["python", "app.py"]
```

_(이 파일을 위해 간단한 `requirements.txt`와 `app.py` 파일도 필요할 수 있습니다)_

---

**2. `Dockerfile.review` (중간 잠재력 / 검토 필요)**

- **의도:** 베이스 이미지는 ARM을 지원하지만, `.so` 파일 복사나 아키텍처 관련 키워드가 포함된 라인 등, 수동 검토가 필요한 요소가 있는 케이스를 테스트합니다.
- **예상 결과:**
  - 베이스 이미지(`ubuntu:22.04`)는 ARM64 지원 (`compatible: True`).
  - `COPY *.so` 라인 감지 및 검토 권장.
  - `"amd64 specific"` 키워드가 포함된 `RUN` 명령어 감지 및 검토 권장.
  - **Overall Potential: Medium**

```dockerfile
# Dockerfile.review: 수동 검토가 필요한 시나리오
FROM ubuntu:22.04

RUN apt-get update && \
    apt-get install -y curl && \
    rm -rf /var/lib/apt/lists/*

# 사전 컴파일된 라이브러리 복사 (ARM 호환성 검증 필요)
COPY my_precompiled_library.so /usr/local/lib/
# COPY helper_script_amd64 /usr/local/bin/ # 이름에 amd64 포함

RUN echo "Checking system architecture..." && uname -m

# 잠재적으로 아키텍처에 따라 다른 동작을 할 수 있는 스크립트 실행
# 또는 단순히 키워드 포함
RUN echo "Applying amd64 specific optimizations if detected..." # 'amd64' 키워드 포함

# 이 라이브러리가 실제로 ARM에서 작동하는지 확인 필요
RUN ldconfig

CMD ["/bin/bash"]
```

_(이 파일을 위해 가상의 `my_precompiled_library.so` 파일이 필요할 수 있습니다)_

---

**3. `Dockerfile.bad` (낮은 ARM 마이그레이션 잠재력)**

- **의도:** ARM64를 지원하지 않는(것으로 가정하는) 베이스 이미지를 사용하거나, 명백한 x86 종속성(바이너리 다운로드, 아키텍처 추가)을 포함하여 마이그레이션이 어려운 케이스를 테스트합니다.
- **예상 결과:**
  - 베이스 이미지(`centos:7` - 실제로는 multi-arch지만, 테스트 목적으로 다른 비호환 이미지로 가정하거나, 여기서 다른 문제가 발견될 것임)에 대한 분석 결과 또는...
  - `dpkg --add-architecture amd64` 감지 (Blocker).
  - `wget ..._amd64.deb` 감지 (Blocker).
  - `yum install ...x86_64` 감지 (Blocker).
  - **Overall Potential: Low**

```dockerfile
# Dockerfile.bad: ARM 마이그레이션 가능성이 낮은 시나리오

# 오래된 OS 버전 또는 특정 아키텍처 이미지를 가정 (CentOS 7은 실제론 multi-arch)
# 만약 이 이미지가 ARM 지원 안한다고 나오거나, 다른 라인에서 문제가 발견될 것
FROM centos:7

# 명백한 x86 종속성 추가 (Debian/Ubuntu 계열 명령어 예시)
# 실제 CentOS에서는 dpkg/apt-get을 쓰지 않지만, 테스트 패턴용으로 포함
RUN echo "Adding amd64 architecture explicitly (for testing pattern)"
# RUN dpkg --add-architecture amd64 && apt-get update

# x86_64 아키텍처용 패키지 설치 시도 (CentOS 방식)
RUN yum install -y epel-release && \
    yum install -y some-package.x86_64 # 명시적인 x86_64 패키지

# 특정 amd64 바이너리 다운로드
RUN curl -L https://example.com/downloads/my_tool_v1.2_amd64.deb -o /tmp/my_tool.deb
# RUN apt-get install -y /tmp/my_tool.deb

WORKDIR /app

COPY . .

CMD ["/app/run.sh"]
```

---

**4. `Dockerfile.multi` (복합적인 케이스)**

- **의도:** 멀티 스테이지 빌드, `scratch` 이미지 사용, 다른 레지스트리(Docker Hub 사용자, GHCR 등) 이미지 사용, 태그 미지정(`latest` 사용) 등 다양한 케이스를 혼합하여 테스트합니다.
- **예상 결과:**
  - `golang:1.19-alpine` (ARM 지원).
  - `scratch` (ARM 지원 - 특별 케이스).
  - `nginx` (`latest` 태그, ARM 지원).
  - `docker.io/bitnami/redis` (Docker Hub 사용자/조직 이미지, ARM 지원 확인 필요).
  - `ghcr.io/octocat/hello-world:latest` (다른 레지스트리, 현재 구현에서는 'unknown'으로 나올 가능성 높음).
  - 각 이미지에 대한 개별 평가와 종합적인 결과가 나와야 함. 잠재력은 'unknown' 이미지 때문에 'Medium' 또는 'Low'가 될 수 있음.

```dockerfile
# Dockerfile.multi: 복합적인 시나리오 (멀티 스테이지, 다른 레지스트리 등)

# --- Build Stage ---
FROM golang:1.19-alpine AS builder

WORKDIR /src
COPY main.go .
# GOARCH는 빌드 환경에 따라 설정될 수 있음
RUN go build -o /app/my_app main.go

# --- Intermediate Stage (다른 레지스트리 예시) ---
FROM docker.io/bitnami/redis:latest AS redis-base
# FROM ghcr.io/octocat/hello-world:latest AS ghcr-example # GHCR 예시 (현재 unknown 예상)

# --- Final Stage ---
FROM scratch

# 빌더에서 바이너리 복사
COPY --from=builder /app/my_app /app/my_app

# 다른 이미지에서 설정 파일 복사 예시 (latest 태그)
FROM nginx
COPY --from=0 /etc/nginx/nginx.conf /etc/nginx/nginx.conf # 이전 스테이지 참조

# 이 라인은 최종 이미지에 영향을 주지 않지만, 분석기는 참조된 nginx 이미지도 검사해야 함

WORKDIR /app
CMD ["/app/my_app"]
```

_(이 파일을 위해 가상의 `main.go` 파일이 필요할 수 있습니다)_

---

**사용 방법:**

1. 위 Dockerfile 예시들을 (`Dockerfile.good`, `Dockerfile.review`, `Dockerfile.bad`, `Dockerfile.multi`) 개인 GitHub 리포지토리에 추가합니다. (필요하다면 간단한 `requirements.txt`, `app.py`, `main.go` 등도 추가하세요.)
2. 해당 리포지토리 URL을 입력으로 사용하여 `DockerAnalyzer`가 포함된 분석 시스템을 실행합니다.
3. 각 Dockerfile에 대해 분석기가 생성하는 `recommendations`, `reasoning`, `overall_potential` 결과를 위에서 설명한 **예상 결과**와 비교하여 분석기가 의도대로 작동하는지 확인합니다.

이 Dockerfile들을 통해 분석기의 다양한 탐지 로직과 평가 기준이 잘 작동하는지 효과적으로 테스트할 수 있을 것입니다.
