# Hello Server - Docker 컨테이너 구축 가이드

## 개요

이 프로젝트는 Go 언어로 작성된 간단한 HTTP 서버를 Docker 컨테이너로 빌드하고 실행하는 방법을 다룹니다.

## 프로젝트 구조

```
hello-server/
├── main.go      # Go HTTP 서버 소스 코드
├── go.mod       # Go 모듈 정의 파일
├── Dockerfile   # Docker 이미지 빌드 설정
└── README.md    # 이 문서
```

## 파일 설명

### main.go

Go 표준 라이브러리를 사용하여 간단한 HTTP 서버를 구현한 파일입니다.

- **포트**: 8080
- **엔드포인트**: `/` (루트 경로)
- **응답**: "Hello, world!"

```go
package main

import (
	"fmt"
	"log"
	"net/http"
)

func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "Hello, world!")
	})

	log.Println("Starting server on port 8080")
	err := http.ListenAndServe(":8080", nil)
	if err != nil {
		log.Fatal(err)
	}
}
```

### go.mod

Go 모듈 정의 파일입니다.

```go
module github.com/make-docker-container

go 1.21
```

- **모듈 이름**: `github.com/make-docker-container`
- **Go 버전**: 1.21

### Dockerfile

멀티 스테이지 빌드를 사용하여 최소 크기의 Docker 이미지를 생성합니다.

```dockerfile
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
ENV CGO_ENABLED=0
RUN go build -o hello .

FROM scratch
COPY --from=builder /app/hello /hello
CMD ["/hello"]
```

**빌드 단계 설명:**

1. **Builder 스테이지** (`golang:1.21`)

   - Go 컴파일러가 포함된 이미지 사용
   - 소스 코드를 복사하고 빌드
   - `CGO_ENABLED=0`: CGO 비활성화로 정적 링크 바이너리 생성

2. **Runtime 스테이지** (`scratch`)
   - 최소 크기의 빈 이미지 사용
   - 빌드된 바이너리만 복사
   - 최종 이미지 크기: 약 6.72MB

## 사전 준비

### 1. Docker Desktop 설치

Windows 환경에서 Docker Desktop을 설치합니다.

### 2. Git Bash에서 Docker 명령어 사용 설정

Git Bash에서 Docker 명령어를 사용하려면 PATH를 설정해야 합니다.

#### 임시 설정 (현재 세션만)

```bash
export PATH="$PATH:/c/Program Files/Docker/Docker/resources/bin"
```

#### 영구 설정

`.bashrc` 파일에 다음 내용을 추가합니다:

```bash
# Docker Desktop PATH
export PATH="$PATH:/c/Program Files/Docker/Docker/resources/bin"
```

파일 저장 후 다음 명령어로 적용:

```bash
source ~/.bashrc
```

또는 한 줄로 추가:

```bash
echo 'export PATH="$PATH:/c/Program Files/Docker/Docker/resources/bin"' >> ~/.bashrc
source ~/.bashrc
```

#### Docker 경로 확인

Docker Desktop의 실제 설치 경로를 확인하려면:

```bash
ls -la "/c/Program Files/Docker/Docker/resources/bin"
```

일반적인 경로:

- `/c/Program Files/Docker/Docker/resources/bin` (기본 설치)
- `/c/Users/[사용자명]/AppData/Local/Docker/resources/bin` (사용자별 설치)

## 빌드 및 실행

### 1. Docker 이미지 빌드

프로젝트 루트 디렉토리에서 실행:

```bash
docker build ./hello-server --tag hello-server:1.0
```

**명령어 설명:**

- `docker build`: Docker 이미지 빌드
- `./hello-server`: 빌드 컨텍스트 (Dockerfile이 있는 디렉토리)
- `--tag hello-server:1.0`: 이미지 이름과 태그 지정

**빌드 결과:**

```
[+] Building 1.1s (10/10) FINISHED
 => [internal] load build definition from Dockerfile
 => [internal] load metadata for docker.io/library/golang:1.21
 => [builder 1/4] FROM docker.io/library/golang:1.21
 => [builder 2/4] WORKDIR /app
 => [builder 3/4] COPY . .
 => [builder 4/4] RUN go build -o hello .
 => [stage-1 1/1] COPY --from=builder /app/hello /hello
 => exporting to image
 => => naming to docker.io/library/hello-server:1.0
```

### 2. 빌드된 이미지 확인

```bash
docker images hello-server
```

**출력 예시:**

```
IMAGE              ID             DISK USAGE   CONTENT SIZE   EXTRA
hello-server:1.0   782f90c7a545       6.72MB             0B
```

### 3. 컨테이너 실행

```bash
docker run --rm --detach --publish 8080:8080 --name hello-server hello-server:1.0
```

**명령어 옵션 설명:**

- `--rm`: 컨테이너 종료 시 자동 삭제
- `--detach` (또는 `-d`): 백그라운드 실행
- `--publish 8080:8080` (또는 `-p 8080:8080`): 호스트 포트 8080을 컨테이너 포트 8080에 매핑
- `--name hello-server`: 컨테이너 이름 지정
- `hello-server:1.0`: 실행할 이미지

**출력:**
컨테이너 ID가 출력됩니다:

```
7feee444260a7ed937946bf2a3da10332aa79c5f0108e7d318166424b283abd4
```

### 4. 서버 테스트

브라우저에서 접속:

```
http://localhost:8080
```

또는 curl 명령어로 테스트:

```bash
curl localhost:8080
```

**응답:**

```
Hello, world!
```

### 5. 컨테이너 중지

```bash
docker stop hello-server
```

## 유용한 Docker 명령어

### 실행 중인 컨테이너 확인

```bash
docker ps
```

### 모든 컨테이너 확인 (중지된 것 포함)

```bash
docker ps -a
```

### 컨테이너 로그 확인

```bash
docker logs hello-server
```

### 컨테이너 내부 접속

```bash
docker exec -it hello-server /bin/sh
```

### 이미지 삭제

```bash
docker rmi hello-server:1.0
```

### 컨테이너 강제 삭제

```bash
docker rm -f hello-server
```

## 트러블슈팅

### 1. "docker: command not found" 오류

**원인**: Git Bash에서 Docker 경로가 설정되지 않음

**해결**: 위의 "Git Bash에서 Docker 명령어 사용 설정" 섹션 참조

### 2. "Cannot connect to the Docker daemon" 오류

**원인**: Docker Desktop이 실행되지 않음

**해결**: Docker Desktop을 실행하고 시스템 트레이에서 실행 중인지 확인

### 3. 포트 충돌 오류

**원인**: 8080 포트가 이미 사용 중

**해결**: 다른 포트 사용

```bash
docker run --rm --detach --publish 8081:8080 --name hello-server hello-server:1.0
```

## 참고사항

- 멀티 스테이지 빌드를 사용하여 최종 이미지 크기를 최소화했습니다
- `scratch` 이미지는 최소 크기이지만 디버깅 도구가 없습니다
- 프로덕션 환경에서는 보안 및 모니터링을 위한 추가 설정이 필요합니다
