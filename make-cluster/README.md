# Kind를 사용한 Kubernetes 클러스터 구축 가이드

## 개요

이 가이드는 Windows 환경에서 Kind (Kubernetes in Docker)를 사용하여 로컬 Kubernetes 클러스터를 생성하고 관리하는 방법을 다룹니다.

## 사전 준비

### 1. Docker Desktop 설치 및 실행

Kind는 Docker를 사용하므로 Docker Desktop이 설치되어 있고 실행 중이어야 합니다.

- Docker Desktop 다운로드: https://www.docker.com/products/docker-desktop
- 설치 후 Docker Desktop을 실행하여 Docker 데몬이 동작하는지 확인

### 2. Kind 설치

#### Windows PowerShell에서 설치

```powershell
# Kind 다운로드 (최신 버전: v0.31.0)
curl.exe -Lo kind-windows-amd64.exe https://kind.sigs.k8s.io/dl/v0.31.0/kind-windows-amd64

# 현재 위치에서 이름 변경 (System32에 있는 경우)
Rename-Item .\kind-windows-amd64.exe kind.exe
```

**참고**: `c:\some-dir-in-your-PATH\kind.exe`는 예시 경로이므로 실제로 존재하지 않습니다.

- 현재 위치(`C:\WINDOWS\system32`)에서 바로 사용하거나
- PATH에 포함된 디렉터리로 이동해야 합니다

#### PATH에 추가 (선택사항)

특정 디렉터리에 설치하려면:

```powershell
# 디렉터리 생성
New-Item -ItemType Directory -Path C:\tools\kind -Force

# 파일 이동
Move-Item .\kind-windows-amd64.exe C:\tools\kind\kind.exe

# PATH에 추가
[Environment]::SetEnvironmentVariable("Path", $env:Path + ";C:\tools\kind", "User")
```

### 3. kubectl 설치

Kubernetes 클러스터를 관리하기 위해 kubectl이 필요합니다.

#### Chocolatey를 사용한 설치

```powershell
choco install kubernetes-cli
```

#### 직접 다운로드

- 공식 문서: https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/
- 또는 winget 사용: `winget install -e --id Kubernetes.kubectl`

## 설치 확인

### Kind 버전 확인

```powershell
.\kind.exe version
```

**출력 예시:**

```
kind v0.31.0 go1.25.5 windows/amd64
```

또는 PATH에 추가된 경우:

```powershell
kind version
```

### kubectl 버전 확인

```powershell
kubectl version --client
```

## 클러스터 생성

### 기본 클러스터 생성

```powershell
kind create cluster
```

이 명령어는:

- 기본 이름 `kind`로 클러스터 생성
- 최신 Kubernetes 버전 사용
- 단일 노드 클러스터 생성

### 특정 Kubernetes 버전으로 클러스터 생성

```powershell
kind create cluster --image=kindest/node:v1.29.0
```

**출력 예시:**

```
Creating cluster "kind" ...
 • Ensuring node image (kindest/node:v1.29.0) 🖼  ...
 ✓ Ensuring node image (kindest/node:v1.29.0) 🖼
 • Preparing nodes 📦   ...
 ✓ Preparing nodes 📦
 • Writing configuration 📜  ...
 ✓ Writing configuration 📜
 • Starting control-plane 🕹️  ...
 ✓ Starting control-plane 🕹️
 • Installing CNI 🔌  ...
 ✓ Installing CNI 🔌
 • Installing StorageClass 💾  ...
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind
```

### 이름을 지정하여 클러스터 생성

```powershell
kind create cluster --name my-cluster
```

여러 클러스터를 동시에 관리할 때 유용합니다.

### 클러스터 생성 오류

**오류: "node(s) already exist for a cluster with the name "kind""**

이미 같은 이름의 클러스터가 존재할 때 발생합니다.

**해결 방법:**

1. 기존 클러스터 삭제 후 재생성

   ```powershell
   kind delete cluster
   kind create cluster
   ```

2. 다른 이름으로 클러스터 생성
   ```powershell
   kind create cluster --name my-cluster
   ```

## 클러스터 확인 및 사용

### 클러스터 정보 확인

```powershell
kubectl cluster-info --context kind-kind
```

**출력 예시:**

```
Kubernetes control plane is running at https://127.0.0.1:57376
CoreDNS is running at https://127.0.0.1:57376/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

### 노드 확인

```powershell
kubectl get nodes
```

**출력 예시:**

```
NAME                 STATUS   ROLES           AGE   VERSION
kind-control-plane   Ready    control-plane   2m    v1.29.0
```

### 모든 리소스 확인

```powershell
kubectl get all --all-namespaces
```

### kubectl 컨텍스트 확인

```powershell
kubectl config get-contexts
```

현재 사용 중인 컨텍스트 확인:

```powershell
kubectl config current-context
```

## 클러스터 관리

### 클러스터 목록 확인

```powershell
kind get clusters
```

### 클러스터 삭제

기본 클러스터 삭제:

```powershell
kind delete cluster
```

**출력 예시:**

```
Deleting cluster "kind" ...
Deleted nodes: ["kind-control-plane"]
```

특정 이름의 클러스터 삭제:

```powershell
kind delete cluster --name my-cluster
```

### kubectl 설정 확인

```powershell
kubectl config view
```

kubectl 설정 파일 위치:

- Windows: `%USERPROFILE%\.kube\config`

## 유용한 명령어

### 클러스터 로그 확인

```powershell
# 컨트롤 플레인 로그
docker logs kind-control-plane

# 특정 컨테이너 로그
docker logs <container-name>
```

### 노드에 접속

```powershell
docker exec -it kind-control-plane bash
```

### 클러스터 내부 네트워크 확인

```powershell
docker network ls
docker network inspect kind
```

### 이미지 로드

로컬 Docker 이미지를 Kind 클러스터에 로드:

```powershell
kind load docker-image hello-server:1.0
```

또는 tar 파일에서:

```powershell
docker save hello-server:1.0 -o hello-server.tar
kind load image-archive hello-server.tar
```

## 실습 예제

### 1. 클러스터 생성 및 확인

```powershell
# 클러스터 생성
kind create cluster --image=kindest/node:v1.29.0

# 클러스터 정보 확인
kubectl cluster-info --context kind-kind

# 노드 확인
kubectl get nodes
```

### 2. 간단한 Pod 배포

```powershell
# nginx Pod 생성
kubectl run nginx --image=nginx

# Pod 상태 확인
kubectl get pods

# Pod 상세 정보
kubectl describe pod nginx
```

### 3. Deployment 생성

```powershell
# Deployment 생성
kubectl create deployment hello-server --image=hello-server:1.0

# Deployment 확인
kubectl get deployments

# Pod 확인
kubectl get pods -l app=hello-server
```

### 4. Service 생성 및 포트 포워딩

```powershell
# Service 생성
kubectl expose deployment hello-server --type=NodePort --port=8080

# Service 확인
kubectl get services

# 포트 포워딩
kubectl port-forward service/hello-server 8080:8080
```

## 트러블슈팅

### 1. "docker: command not found" 오류

**원인**: Docker Desktop이 실행되지 않음

**해결**: Docker Desktop을 실행하고 시스템 트레이에서 실행 중인지 확인

### 2. "Cannot connect to the Docker daemon" 오류

**원인**: Docker 데몬이 실행되지 않음

**해결**:

- Docker Desktop 재시작
- Windows 서비스에서 Docker Desktop 서비스 확인

### 3. "node(s) already exist" 오류

**원인**: 같은 이름의 클러스터가 이미 존재

**해결**:

```powershell
kind delete cluster
kind create cluster
```

### 4. kubectl이 클러스터에 연결되지 않음

**원인**: kubectl 컨텍스트가 설정되지 않음

**해결**:

```powershell
kubectl config use-context kind-kind
```

### 5. 포트 충돌

**원인**: 다른 애플리케이션이 동일한 포트 사용

**해결**:

- 다른 포트 사용
- 충돌하는 프로세스 종료

## 참고사항

- Kind는 로컬 개발 및 테스트용으로 설계되었습니다
- 프로덕션 환경에서는 사용하지 마세요
- 클러스터는 Docker 컨테이너로 실행되므로 Docker Desktop이 필요합니다
- 여러 클러스터를 동시에 실행할 수 있습니다 (각각 다른 이름 사용)
- 클러스터 삭제 시 모든 데이터가 삭제됩니다

## 추가 리소스

- Kind 공식 문서: https://kind.sigs.k8s.io/
- Kubernetes 공식 문서: https://kubernetes.io/docs/
- kubectl 치트시트: https://kubernetes.io/docs/reference/kubectl/cheatsheet/
