# Pod 생성 실습 가이드

## 개요

이 가이드는 Kubernetes Pod를 생성하고 관리하는 실습을 다룹니다. Windows Git Bash 환경에서 kind 클러스터에 Pod를 배포하는 과정을 설명합니다.

## 사전 준비

### 1. Kind 클러스터 생성

Pod를 배포하기 전에 kind 클러스터가 생성되어 있어야 합니다.

```bash
# 프로젝트 루트에서 환경 설정 적용
source .bashrc

# 클러스터 생성
kind create cluster --image=kindest/node:v1.29.0

# 클러스터 확인
kubectl cluster-info --context kind-kind
kubectl get nodes
```

자세한 내용은 [make-cluster 가이드](../make-cluster/README.md)를 참조하세요.

### 2. 프로젝트 환경 설정 (.bashrc 파일 생성 및 사용)

이 실습에서는 프로젝트 루트에 `.bashrc` 파일을 생성하여 docker와 kubectl을 자동으로 인식할 수 있도록 PATH를 설정합니다.

#### .bashrc 파일이란?

`.bashrc`는 Git Bash(또는 Linux/Mac의 bash)가 시작될 때 자동으로 실행되는 설정 파일입니다. 프로젝트 루트에 `.bashrc` 파일을 만들면 프로젝트 전용 설정을 관리할 수 있습니다.

**장점:**

- 프로젝트별로 독립적인 설정 관리
- 프로젝트 삭제 시 설정도 자동으로 제거
- 다른 프로젝트에 영향을 주지 않음

#### .bashrc 파일 생성

프로젝트 루트 디렉토리에 `.bashrc` 파일을 생성합니다:

```bash
# 프로젝트 루트에서
cat > .bashrc << 'EOF'
# Docker 경로 추가
export PATH="/c/Program Files/Docker/Docker/resources/bin:$PATH"

# kubectl 경로 추가
export PATH="/c/ProgramData/chocolatey/bin:$PATH"
EOF
```

또는 직접 파일을 생성:

```bash
# .bashrc 파일 생성
echo '# Docker 경로 추가' > .bashrc
echo 'export PATH="/c/Program Files/Docker/Docker/resources/bin:$PATH"' >> .bashrc
echo '' >> .bashrc
echo '# kubectl 경로 추가' >> .bashrc
echo 'export PATH="/c/ProgramData/chocolatey/bin:$PATH"' >> .bashrc
```

**파일 내용 확인:**

```bash
cat .bashrc
```

**출력 예시:**

```
# Docker 경로 추가
export PATH="/c/Program Files/Docker/Docker/resources/bin:$PATH"

# kubectl 경로 추가
export PATH="/c/ProgramData/chocolatey/bin:$PATH"
```

#### .bashrc 파일 사용 방법

프로젝트 루트 디렉토리에서:

```bash
source .bashrc
```

이 명령어는 현재 세션에 `.bashrc` 파일의 설정을 적용합니다.

#### PATH 환경 변수 상세 설명

**PATH란?**

- PATH는 시스템이 실행 파일을 찾는 경로 목록입니다
- 명령어를 입력하면 시스템이 PATH에 있는 경로들을 순서대로 검색합니다
- 예: `docker` 입력 → PATH에서 `docker.exe` 찾기 → 실행

**export PATH 명령어 분석:**

```bash
export PATH="/c/Program Files/Docker/Docker/resources/bin:$PATH"
```

**각 부분 설명:**

1. **`export`**: 환경 변수를 현재 세션과 하위 프로세스에서 사용 가능하게 설정

   - `export` 없이 `PATH=...`만 하면 현재 셸에서만 적용
   - `export`를 사용하면 이 셸에서 실행하는 모든 프로그램이 이 PATH를 사용

2. **`PATH=`**: PATH 변수에 값을 할당

3. **`"/c/Program Files/Docker/Docker/resources/bin"`**: 추가할 경로

   - Windows 경로 `C:\Program Files\...`를 Git Bash 형식으로 변환
   - 변환 규칙:
     - `C:\` → `/c/`
     - 백슬래시(`\`) → 슬래시(`/`)
     - 공백이 있으므로 따옴표로 감싸야 함

4. **`:$PATH`**: 기존 PATH 값 앞에 새 경로를 추가
   - `:`는 경로 구분자
   - `$PATH`는 기존 PATH 값
   - 새 경로를 앞에 추가하므로 우선순위가 높음

**실제 동작 예시:**

```bash
# 기존 PATH
/mingw64/bin:/usr/bin:/bin

# 명령어 실행 후
/c/Program Files/Docker/Docker/resources/bin:/mingw64/bin:/usr/bin:/bin
```

#### Windows 경로를 Git Bash 형식으로 변환

**변환 규칙:**

| Windows 경로           | Git Bash 경로          |
| ---------------------- | ---------------------- |
| `C:\Program Files\...` | `/c/Program Files/...` |
| `C:\Users\admin\...`   | `/c/Users/admin/...`   |
| `C:\Windows\...`       | `/c/Windows/...`       |

**변환 방법:**

1. 드라이브 문자를 소문자로 변환: `C:` → `c`
2. 드라이브 문자 앞에 `/` 추가: `c` → `/c`
3. 백슬래시를 슬래시로 변환: `\` → `/`
4. 공백이 있으면 따옴표로 감싸기

#### docker와 kubectl 경로 확인

**Docker 경로 확인:**

```bash
ls -la "/c/Program Files/Docker/Docker/resources/bin/docker.exe"
```

**kubectl 경로 확인:**

```bash
which kubectl
# 또는
ls -la "/c/ProgramData/chocolatey/bin/kubectl"
```

#### 설정 확인 방법

`.bashrc`를 적용한 후 다음 명령어로 확인:

```bash
# Docker 경로 확인
which docker
# 출력 예시: /c/Program Files/Docker/Docker/resources/bin/docker

# kubectl 경로 확인
which kubectl
# 출력 예시: /c/ProgramData/chocolatey/bin/kubectl

# PATH 전체 확인
echo $PATH
# Docker와 kubectl 경로가 포함되어 있는지 확인

# 명령어 작동 확인
docker --version
kubectl version --client
```

#### 오류 해결

**오류 예시:**

```
ERROR: failed to create cluster: failed to get docker info: command "docker info --format '{{json .}}'" failed with error: exec: "docker": executable file not found in %PATH%
```

**원인:**

- Docker 경로가 PATH에 없음
- Git Bash가 Windows PATH를 제대로 상속하지 못함

**해결 방법:**

1. 프로젝트 `.bashrc` 사용 (권장):

   ```bash
   source .bashrc
   ```

2. 수동으로 PATH 추가:

   ```bash
   export PATH="/c/Program Files/Docker/Docker/resources/bin:$PATH"
   ```

3. Docker 경로 확인:

   ```bash
   ls -la "/c/Program Files/Docker/Docker/resources/bin/docker.exe"
   ```

4. kubectl 경로 확인:

   ```bash
   which kubectl
   # 또는
   ls -la "/c/ProgramData/chocolatey/bin/kubectl"
   ```

#### 실습 단계 요약

1. ✅ 프로젝트 루트 디렉토리로 이동
2. ✅ `.bashrc` 파일 생성 (docker와 kubectl 경로 추가)
3. ✅ `source .bashrc`로 설정 적용
4. ✅ `which docker`, `which kubectl`로 경로 확인
5. ✅ `docker --version`, `kubectl version --client`로 작동 확인

## Pod 생성 실습

### 1. Pod 매니페스트 파일 확인

`myapp.yaml` 파일 내용:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  containers:
    - name: hello-server
      image: blux2/hello-server:1.0
      ports:
        - containerPort: 8080
```

### 2. Pod 생성

```bash
# make-the-pod 디렉토리로 이동
cd make-the-pod

# Pod 생성
kubectl apply -f myapp.yaml
```

**출력 예시:**

```
pod/myapp created
```

### 3. Pod 상태 확인

```bash
# Pod 목록 확인
kubectl get pod

# 상세 정보 확인
kubectl describe pod myapp

# Pod 로그 확인
kubectl logs myapp
```

**출력 예시:**

```
NAME    READY   STATUS    RESTARTS   AGE
myapp   1/1     Running   0          13s
```

### 4. Pod 접속 테스트

```bash
# Pod 내부에서 명령어 실행
kubectl exec -it myapp -- /bin/sh

# 또는 curl로 서비스 테스트 (포트 포워딩 필요)
kubectl port-forward pod/myapp 8080:8080
```

다른 터미널에서:

```bash
curl localhost:8080
```

## 주요 명령어

### Pod 관리

```bash
# Pod 생성
kubectl apply -f myapp.yaml

# Pod 목록 확인
kubectl get pods
kubectl get pod

# Pod 상세 정보
kubectl describe pod <pod-name>

# Pod 로그 확인
kubectl logs <pod-name>
kubectl logs -f <pod-name>  # 실시간 로그

# Pod 삭제
kubectl delete pod <pod-name>
kubectl delete -f myapp.yaml
```

### 디버깅

```bash
# Pod 내부 쉘 접속
kubectl exec -it <pod-name> -- /bin/sh

# 특정 컨테이너 접속 (멀티 컨테이너인 경우)
kubectl exec -it <pod-name> -c <container-name> -- /bin/sh

# Pod 이벤트 확인
kubectl get events --sort-by='.lastTimestamp'

# Pod YAML 출력
kubectl get pod <pod-name> -o yaml
```

## 문제 해결

### 1. "docker: executable file not found" 오류

**원인:** Git Bash에서 Docker 경로를 찾을 수 없음

**해결:**

```bash
export PATH="/c/Program Files/Docker/Docker/resources/bin:$PATH"
```

### 2. Pod가 Pending 상태로 유지됨

**원인:** 리소스 부족 또는 이미지 다운로드 실패

**확인:**

```bash
kubectl describe pod myapp
kubectl get events
```

### 3. Pod가 CrashLoopBackOff 상태

**원인:** 컨테이너가 계속 재시작됨

**확인:**

```bash
kubectl logs myapp
kubectl describe pod myapp
```

### 4. 이미지를 찾을 수 없음

**원인:** 이미지가 존재하지 않거나 접근 불가

**해결:**

- 이미지 이름 확인
- 이미지가 Docker Hub에 있는지 확인
- 필요시 이미지를 직접 빌드하여 kind 클러스터에 로드

```bash
# 로컬 이미지를 kind 클러스터에 로드
kind load docker-image hello-server:1.0
```

## 실습 단계 요약

1. ✅ Kind 클러스터 생성 확인
2. ✅ Git Bash에서 Docker PATH 설정
3. ✅ Pod 매니페스트 파일 작성 (`myapp.yaml`)
4. ✅ `kubectl apply -f myapp.yaml`로 Pod 생성
5. ✅ `kubectl get pod`로 상태 확인
6. ✅ Pod가 Running 상태가 될 때까지 대기
7. ✅ `kubectl logs`로 로그 확인
8. ✅ `kubectl exec`로 Pod 내부 접속 테스트

## 다음 단계

- [Deployment 실습](../README.md) - Pod를 관리하는 더 고수준 리소스
- [Service 실습](../README.md) - Pod에 네트워크 접근 제공
- [ConfigMap과 Secret](../README.md) - 설정 및 민감 정보 관리

## 참고 자료

- [Kubernetes Pod 문서](https://kubernetes.io/docs/concepts/workloads/pods/)
- [kubectl 치트시트](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [Kind 공식 문서](https://kind.sigs.k8s.io/)
