# Kubernetes 리소스 만들고 망가뜨리기 실습 실행 정리

## 개요

이 문서는 Kubernetes Deployment와 Service를 생성하고, 의도적으로 문제를 발생시킨 후 해결하는 실습 실행 내용을 정리합니다. Rolling Update 중 이미지 오류 발생 및 NodePort Service 접근 문제를 해결하는 과정을 기록합니다.

## 사전 준비

### 1. 이전 실습 완료

이 실습을 진행하기 전에 다음 실습을 완료해야 합니다:

- [make-cluster 실행 정리](../make-cluster/README.md) - Kind 클러스터 생성
- [make-the-pod 실행 정리](../make-the-pod/README.md) - Pod 생성 실습
- [try-debugging 실행 정리](../try-debugging/README.md) - Pod 디버깅 실습

### 2. 환경 설정

```bash
# 프로젝트 루트에서 환경 설정 적용
source .bashrc

# kubectl alias 설정
alias k=kubectl
```

## 실습 파일

### deployment-hello-server.yaml

정상적인 Deployment 설정:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-server
  labels:
    app: hello-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello-server
  template:
    metadata:
      labels:
        app: hello-server
    spec:
      containers:
        - name: hello-server
          image: blux2/hello-server:1.0
          ports:
            - containerPort: 8080
```

### deployment-hello-server-rollingupdate.yaml

문제가 있는 Deployment 설정 (존재하지 않는 이미지 버전):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-server
  labels:
    app: hello-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello-server
  template:
    metadata:
      labels:
        app: hello-server
    spec:
      containers:
        - name: hello-server
          image: blux2/hello-server:1.3 # 존재하지 않는 버전
          ports:
            - containerPort: 8080
```

### service-nodeport.yaml

NodePort Service 설정:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-server-external
spec:
  type: NodePort
  selector:
    app: hello-server
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 30599
```

### kind-config-portmapping.yaml

포트 매핑이 포함된 Kind 클러스터 설정:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 30599
        hostPort: 30599
        listenAddress: "127.0.0.1"
        protocol: TCP
```

## 실습 단계

### 1. Deployment 생성

```bash
# break-and-fix 디렉토리로 이동
cd break-and-fix

# Deployment 생성
kubectl apply -f deployment-hello-server.yaml
```

**출력 예시:**

```
deployment.apps/hello-server created
```

**Pod 상태 확인:**

```bash
kubectl get pods -l app=hello-server
```

**출력 예시:**

```
NAME                            READY   STATUS    RESTARTS   AGE
hello-server-6cc6b44795-8lwbm   1/1     Running   0          8s
hello-server-6cc6b44795-bn86x   1/1     Running   0          8s
hello-server-6cc6b44795-qzcfs   1/1     Running   0          10s
```

### 2. Pod 삭제 테스트 (자동 복구 확인)

```bash
# Pod 하나 삭제
kubectl delete pod hello-server-6cc6b44795-8lwbm

# Pod 상태 확인 (자동으로 새 Pod 생성됨)
kubectl get pods -l app=hello-server
```

**출력 예시:**

```
NAME                            READY   STATUS    RESTARTS   AGE
hello-server-6cc6b44795-bn86x   1/1     Running   0          40s
hello-server-6cc6b44795-qzcfs   1/1     Running   0          40s
hello-server-6cc6b44795-t67bq   1/1     Running   0          6s  # 새로 생성됨
```

**학습 포인트:**

- Deployment는 `replicas: 3`을 유지하기 위해 Pod가 삭제되면 자동으로 새 Pod를 생성합니다
- ReplicaSet이 이를 관리합니다

### 3. Rolling Update 실습 (문제 발생)

**문제가 있는 Deployment 적용:**

```bash
# 존재하지 않는 이미지 버전(1.3)으로 업데이트
kubectl apply -f deployment-hello-server-rollingupdate.yaml
```

**출력 예시:**

```
deployment.apps/hello-server configured
```

**Pod 상태 확인:**

```bash
kubectl get pods -w
```

**출력 예시:**

```
NAME                            READY   STATUS         RESTARTS   AGE
hello-server-6cc6b44795-bn86x   1/1     Running        0          10m
hello-server-6cc6b44795-qzcfs   1/1     Running        0          10m
hello-server-6cc6b44795-t67bq   1/1     Running        0          9m29s
hello-server-6fb85ff748-s464r   0/1     ErrImagePull   0          6s
hello-server-6fb85ff748-s464r   0/1     ImagePullBackOff   0          16s
```

**문제 분석:**

- 새로운 ReplicaSet이 생성되었지만 Pod가 `ErrImagePull` 상태
- 이미지 `blux2/hello-server:1.3`이 존재하지 않음
- 기존 Pod들은 정상 실행 중 (Rolling Update는 점진적으로 진행)

**Deployment 상태 확인:**

```bash
kubectl get deploy hello-server
```

**출력 예시:**

```
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
hello-server   3/3     1            3           10m
```

- `READY: 3/3` - 3개 Pod가 준비됨 (기존 Pod들)
- `UP-TO-DATE: 1` - 1개 Pod가 새 버전으로 업데이트 시도 중 (실패)
- `AVAILABLE: 3` - 3개 Pod가 서비스 가능 (기존 Pod들)

**ReplicaSet 확인:**

```bash
kubectl get replicaset
```

**출력 예시:**

```
NAME                      DESIRED   CURRENT   READY   AGE
hello-server-6cc6b44795   3         3         3       28m  # 기존 버전
hello-server-6fb85ff748   1         1         0       18m  # 새 버전 (실패)
```

### 4. 문제 해결 (Deployment 수정)

**방법 1: kubectl edit 사용**

```bash
# Deployment 편집
kubectl edit deploy hello-server

# 이미지 버전을 1.3 → 1.2로 수정 (또는 존재하는 버전으로)
```

**방법 2: YAML 파일 수정 후 적용**

```bash
# deployment-hello-server.yaml 파일에서 이미지 버전 수정
# image: blux2/hello-server:1.0 (또는 존재하는 버전)

# 적용
kubectl apply -f deployment-hello-server.yaml
```

**수정 후 확인:**

```bash
kubectl get pods -w
```

**출력 예시:**

```
NAME                            READY   STATUS    RESTARTS   AGE
pod/hello-server-5d6fd6dbb9-gwptg   1/1     Running   0          16s
pod/hello-server-5d6fd6dbb9-mpwxp   1/1     Running   0          24s
pod/hello-server-5d6fd6dbb9-r9xz7   1/1     Running   0          17s
```

**ReplicaSet 확인:**

```bash
kubectl get replicaset
```

**출력 예시:**

```
NAME                      DESIRED   CURRENT   READY   AGE
hello-server-5d6fd6dbb9   3         3         3       24s  # 새 버전 (성공)
hello-server-6cc6b44795   0         0         0       30m  # 기존 버전 (스케일 다운)
hello-server-6fb85ff748   0         0         0       20m  # 실패한 버전 (스케일 다운)
```

**학습 포인트:**

- Rolling Update는 점진적으로 진행됩니다
- 새 버전이 실패해도 기존 버전은 계속 서비스됩니다
- 문제를 해결하면 새 버전으로 정상 업데이트됩니다

### 5. NodePort Service 생성

**Service 생성:**

```bash
kubectl apply -f service-nodeport.yaml
```

**출력 예시:**

```
service/hello-server-external created
```

**Service 상태 확인:**

```bash
kubectl get svc hello-server-external
```

**출력 예시:**

```
NAME                    TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
hello-server-external   NodePort   10.96.30.148   <none>        8080:30599/TCP   7m18s
```

**Endpoints 확인:**

```bash
kubectl get endpoints hello-server-external
```

**출력 예시:**

```
NAME                    ENDPOINTS                                            AGE
hello-server-external   10.244.0.13:8080,10.244.0.14:8080,10.244.0.15:8080   7m55s
```

### 6. Service 접근 문제 발생 및 해결

#### 문제: 포트 포워딩 없이 접근 불가

**접근 시도:**

```bash
# NodePort로 직접 접근 시도
curl localhost:30599
```

**결과:**

```
curl: (7) Failed to connect to localhost port 30599 after 2228 ms: Couldn't connect to server
```

#### 인과 관계 분석: 왜 안 됐는가?

**1단계: 문제 상황**

- NodePort Service는 정상적으로 생성됨 (`kubectl get svc` 확인)
- Pod도 정상 실행 중 (`kubectl get pods` 확인)
- Endpoints도 정상 (`kubectl get endpoints` 확인)
- 하지만 `curl localhost:30599` 실패

**2단계: 근본 원인 파악**

kind의 구조적 특성:

```
호스트(Windows)
  ↓
Docker 컨테이너 (kind-control-plane)
  ↓
Kubernetes 클러스터 (컨테이너 내부)
  ↓
NodePort Service (30599 포트)
```

**문제의 핵심:**

- kind는 Docker 컨테이너 내부에서 Kubernetes를 실행함
- Docker 컨테이너의 포트는 기본적으로 호스트에 매핑되지 않음
- **예외**: API 서버 포트(6443)만 자동으로 매핑됨 (kubectl 접근용)

**확인:**

```bash
# Docker 컨테이너 포트 매핑 확인
docker ps | grep kind-control-plane
docker port kind-control-plane
```

**출력 예시 (문제 상황):**

```
6443/tcp -> 127.0.0.1:58631  # API 서버 포트만 매핑됨
# 30599 포트는 매핑되지 않음!
```

**인과 관계:**

```
원인: Docker 컨테이너 포트가 호스트에 매핑되지 않음
  ↓
결과: 호스트에서 localhost:30599로 접근 시도
  ↓
문제: Docker가 해당 포트를 리스닝하지 않음
  ↓
실패: "Failed to connect" 오류 발생
```

**3단계: 왜 이렇게 설계되었는가?**

- kind는 개발/테스트용 도구
- 보안상 모든 포트를 자동 매핑하지 않음
- 필요한 포트만 명시적으로 매핑하도록 설계
- 클러스터 생성 시에만 포트 매핑 설정 가능 (런타임 변경 불가)

#### 해결: 포트 매핑 설정으로 클러스터 재생성

**인과 관계: 어떻게 해결했는가?**

**해결 방법의 핵심:**

- Docker 컨테이너 생성 시 포트 매핑을 명시적으로 설정
- kind는 `extraPortMappings` 설정을 통해 포트 매핑 가능
- **중요**: 클러스터 생성 시에만 설정 가능 (이미 생성된 클러스터에는 적용 불가)

**1. 포트 매핑 설정 파일 생성**

`kind-config-portmapping.yaml` 파일 생성:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 30599 # 컨테이너 내부 포트
        hostPort: 30599 # 호스트 포트
        listenAddress: "127.0.0.1" # 로컬호스트에서만 접근
        protocol: TCP
```

**설정의 의미:**

- `containerPort: 30599`: kind 컨테이너 내부에서 리스닝할 포트
- `hostPort: 30599`: 호스트(Windows)에서 접근할 포트
- `listenAddress: "127.0.0.1"`: 보안을 위해 로컬호스트에서만 접근 허용

**인과 관계:**

```
설정: extraPortMappings에 30599 포트 추가
  ↓
동작: Docker가 컨테이너 생성 시 포트 매핑 설정
  ↓
결과: 호스트의 30599 포트 → 컨테이너의 30599 포트로 연결
  ↓
효과: 호스트에서 localhost:30599 접근 시 컨테이너 내부로 전달
```

**2. 클러스터 재생성 (필수)**

**왜 재생성이 필요한가?**

- 포트 매핑은 컨테이너 생성 시에만 설정 가능
- 이미 실행 중인 컨테이너의 포트 매핑은 변경 불가
- 따라서 클러스터를 삭제하고 새로 생성해야 함

```bash
# 기존 클러스터 삭제
kind delete cluster --name kind

# 포트 매핑 설정으로 클러스터 생성
kind create cluster --config kind-config-portmapping.yaml --image=kindest/node:v1.29.0
```

**인과 관계:**

```
기존 클러스터: 포트 매핑 없음 → 접근 불가
  ↓
클러스터 삭제: 기존 설정 제거
  ↓
새 클러스터 생성: 포트 매핑 설정 포함 → 접근 가능
```

**3. 리소스 재생성**

클러스터를 재생성하면 모든 리소스가 삭제되므로 다시 생성해야 함:

```bash
cd break-and-fix

# Deployment 재생성
kubectl apply -f deployment-hello-server.yaml

# Service 재생성
kubectl apply -f service-nodeport.yaml

# Pod 준비 대기
kubectl wait --for=condition=ready pod -l app=hello-server --timeout=60s
```

**4. 포트 매핑 확인**

```bash
docker port kind-control-plane
```

**출력 예시 (해결 후):**

```
6443/tcp -> 127.0.0.1:63435
30599/tcp -> 127.0.0.1:30599  # ✅ NodePort 매핑됨!
```

**인과 관계:**

```
설정 적용: extraPortMappings에 30599 추가
  ↓
Docker 동작: 컨테이너 생성 시 포트 매핑 설정
  ↓
확인 결과: docker port 명령어로 매핑 확인 가능
```

**5. 접근 테스트**

```bash
curl localhost:30599
```

**출력 예시:**

```
Hello, world!
```

**성공!** 포트 포워딩 없이 직접 접근 가능합니다.

**6. 대안 방법: 컨테이너 내부에서 접근**

포트 매핑 설정 없이도 컨테이너 내부에서 직접 접근할 수 있습니다.

**컨테이너 내부로 접속:**

```bash
# kind 컨테이너 내부로 접속
docker exec -it kind-control-plane bash
```

**명령어 설명:**

- `docker exec`: 실행 중인 컨테이너에서 명령어 실행
- `-it`: 인터랙티브 모드 (터미널 입력/출력 가능)
- `kind-control-plane`: 컨테이너 이름
- `bash`: 실행할 명령어 (bash 쉘)

**컨테이너 내부에서 접근:**

컨테이너 내부 쉘에서:

```bash
# NodePort로 직접 접근
curl localhost:30599

# 또는 노드 IP로 접근
curl 172.18.0.2:30599

# 또는 Service의 ClusterIP로 접근
curl 10.96.30.148:8080
```

**실제 출력:**

```
Hello, world!
```

**한 줄로 실행 (컨테이너 내부로 들어가지 않고):**

```bash
# 컨테이너 내부에서 curl 실행
docker exec kind-control-plane curl localhost:30599
```

**실제 출력:**

```
Hello, world!
```

**왜 가능한가?**

- 컨테이너 내부에서는 포트 매핑 없이도 접근 가능
- Kubernetes NodePort Service가 컨테이너 내부의 30599 포트를 리스닝 중
- 같은 네트워크 안에 있으므로 직접 접근 가능

**비교:**

| 방법                   | 호스트에서 접근 | 컨테이너 내부 접근 | 포트 매핑 필요 |
| ---------------------- | --------------- | ------------------ | -------------- |
| **포트 매핑 설정**     | ✅ 가능         | ✅ 가능            | ✅ 필요        |
| **컨테이너 내부 접속** | ❌ 불가         | ✅ 가능            | ❌ 불필요      |

**언제 사용하나?**

- 포트 매핑 설정 없이 빠르게 테스트하고 싶을 때
- 컨테이너 내부에서 직접 디버깅할 때
- 호스트에서 접근할 필요가 없을 때

#### 최종 인과 관계 정리

**문제 발생:**

```
NodePort Service 생성
  ↓
호스트에서 localhost:30599 접근 시도
  ↓
Docker 포트 매핑 없음
  ↓
접근 실패 ("Failed to connect")
```

**해결 과정:**

```
포트 매핑 설정 파일 생성 (extraPortMappings)
  ↓
클러스터 재생성 (설정 적용)
  ↓
Docker가 포트 매핑 설정 (30599:30599)
  ↓
호스트에서 localhost:30599 접근
  ↓
Docker가 요청을 컨테이너로 전달
  ↓
컨테이너 내부 NodePort Service가 요청 수신
  ↓
Service가 Pod로 트래픽 전달
  ↓
접근 성공 ("Hello, world!")
```

**핵심 인과 관계:**

1. **원인**: Docker 컨테이너 포트가 호스트에 매핑되지 않음
2. **해결**: `extraPortMappings` 설정으로 포트 매핑 명시
3. **조건**: 클러스터 생성 시에만 설정 가능 (재생성 필요)
4. **결과**: 호스트 → Docker → Kubernetes → Pod 연결 완성

## 작동 흐름 및 인과 관계

### 전체 흐름

```
1. Deployment 생성
   ↓
2. Pod 자동 복구 확인 (Pod 삭제 → 자동 재생성)
   ↓
3. Rolling Update 시도 (이미지 버전 1.3)
   ↓
4. 문제 발생 (ErrImagePull - 이미지 없음)
   ↓
5. 기존 Pod는 정상 동작 (서비스 중단 없음)
   ↓
6. Deployment 수정 (존재하는 이미지 버전으로 변경)
   ↓
7. Rolling Update 성공 (새 버전으로 점진적 업데이트)
   ↓
8. NodePort Service 생성
   ↓
9. 접근 시도 → 실패 (포트 매핑 없음)
   ↓
10. 원인 분석 (kind 환경 제약사항)
    ↓
11. 포트 매핑 설정 파일 생성
    ↓
12. 클러스터 재생성 (포트 매핑 포함)
    ↓
13. 리소스 재생성
    ↓
14. 접근 성공 (포트 포워딩 없이 직접 접근 가능)
```

### 인과 관계

**문제 1: Rolling Update 실패**

- **원인**: 존재하지 않는 이미지 버전(`blux2/hello-server:1.3`) 사용
- **결과**: 새 Pod가 `ErrImagePull` 상태로 실패
- **영향**: 기존 Pod는 정상 동작 (서비스 중단 없음)
- **해결**: 존재하는 이미지 버전으로 수정 → Rolling Update 성공

**문제 2: NodePort Service 접근 불가**

- **원인**: kind 환경에서 NodePort가 호스트로 자동 매핑되지 않음
- **결과**: `localhost:30599`로 접근 불가
- **영향**: 포트 포워딩 없이는 외부 접근 불가
- **해결**: `extraPortMappings` 설정으로 클러스터 재생성 → 포트 매핑 생성 → 접근 가능

### 핵심 학습 포인트

1. **Deployment의 자동 복구**

   - Pod 삭제 시 자동으로 새 Pod 생성
   - ReplicaSet이 `replicas` 수를 유지

2. **Rolling Update의 안전성**

   - 점진적으로 업데이트 진행
   - 새 버전 실패 시 기존 버전 유지
   - 서비스 중단 없이 업데이트 가능

3. **kind 환경의 제약사항**

   - NodePort가 기본적으로 호스트에 매핑되지 않음
   - `extraPortMappings` 설정 필요
   - 클러스터 생성 시에만 설정 가능

4. **문제 해결 과정**
   - 문제 발생 → 원인 분석 → 해결 방법 찾기 → 적용 → 검증

## 주요 명령어

### Deployment 관리

```bash
# Deployment 생성
kubectl apply -f deployment-hello-server.yaml

# Deployment 상태 확인
kubectl get deploy hello-server

# Deployment 상세 정보
kubectl describe deploy hello-server

# Deployment 수정
kubectl edit deploy hello-server

# Deployment 삭제
kubectl delete deploy hello-server
```

### Pod 관리

```bash
# Pod 목록 확인
kubectl get pods -l app=hello-server

# Pod 상세 정보
kubectl describe pod <pod-name>

# Pod 삭제 (자동 복구 확인)
kubectl delete pod <pod-name>

# Pod 로그 확인
kubectl logs <pod-name>
```

### ReplicaSet 관리

```bash
# ReplicaSet 목록 확인
kubectl get replicaset

# ReplicaSet 상세 정보
kubectl describe replicaset <replicaset-name>

# ReplicaSet 스케일 조정
kubectl scale replicaset <replicaset-name> --replicas=2
```

### Service 관리

```bash
# Service 생성
kubectl apply -f service-nodeport.yaml

# Service 목록 확인
kubectl get svc

# Service 상세 정보
kubectl describe svc hello-server-external

# Endpoints 확인
kubectl get endpoints hello-server-external

# Service 삭제
kubectl delete svc hello-server-external
```

### Kind 클러스터 관리

```bash
# 클러스터 목록 확인
kind get clusters

# 포트 매핑 설정으로 클러스터 생성
kind create cluster --config kind-config-portmapping.yaml --image=kindest/node:v1.29.0

# 클러스터 삭제
kind delete cluster --name kind

# 포트 매핑 확인
docker port kind-control-plane
```

### Docker 컨테이너 접속

```bash
# kind 컨테이너 내부로 접속 (인터랙티브 모드)
docker exec -it kind-control-plane bash

# 컨테이너 내부에서 명령어 실행 (한 줄로)
docker exec kind-control-plane curl localhost:30599

# 컨테이너 내부에서 kubectl 사용 (설치되어 있는 경우)
docker exec -it kind-control-plane kubectl get pods

# 컨테이너 상태 확인
docker ps | grep kind-control-plane

# 컨테이너 로그 확인
docker logs kind-control-plane
```

**명령어 설명:**

- `docker exec`: 실행 중인 컨테이너에서 명령어 실행
- `-it`: 인터랙티브 모드 (터미널 입력/출력 가능)
- `kind-control-plane`: kind 컨테이너 이름
- `bash`: 실행할 명령어 (bash 쉘)

**사용 예시:**

```bash
# 컨테이너 내부로 접속
docker exec -it kind-control-plane bash

# 컨테이너 내부에서 (포트 매핑 없이도 접근 가능)
curl localhost:30599
# 출력: Hello, world!

# 컨테이너 내부에서 노드 IP로 접근
curl 172.18.0.2:30599
# 출력: Hello, world!

# 컨테이너 내부에서 Service ClusterIP로 접근
curl 10.96.30.148:8080
# 출력: Hello, world!

# 컨테이너 내부에서 나가기
exit
```

**왜 컨테이너 내부에서 접근이 가능한가?**

- 컨테이너 내부에서는 포트 매핑 없이도 접근 가능
- Kubernetes NodePort Service가 컨테이너 내부의 30599 포트를 리스닝 중
- 같은 네트워크 안에 있으므로 직접 접근 가능
- 호스트에서 접근하려면 포트 매핑이 필요하지만, 컨테이너 내부에서는 불필요

## 문제 해결

### 1. ErrImagePull / ImagePullBackOff

**증상:**

```
STATUS: ErrImagePull 또는 ImagePullBackOff
```

**원인:**

- 이미지가 존재하지 않음
- 이미지 이름 또는 태그 오류

**해결:**

```bash
# Pod 상세 정보 확인
kubectl describe pod <pod-name>

# 이미지 버전 확인 및 수정
kubectl edit deploy hello-server
# 또는
kubectl set image deploy/hello-server hello-server=<correct-image>
```

### 2. NodePort Service 접근 불가

**증상:**

```
curl localhost:30599
# Failed to connect
```

**원인:**

- kind 환경에서 포트 매핑이 없음

**해결:**

1. 포트 매핑 설정 파일 생성
2. 클러스터 재생성
3. 리소스 재생성

### 3. Rolling Update가 진행되지 않음

**원인:**

- 새 이미지가 동일한 해시를 가짐
- Deployment 설정이 변경되지 않음

**해결:**

```bash
# 강제로 롤아웃 트리거
kubectl rollout restart deploy hello-server

# 또는 이미지 버전 변경
kubectl set image deploy/hello-server hello-server=<new-image>
```

## 실습 단계 요약

1. ✅ Deployment 생성 및 Pod 자동 복구 확인
2. ✅ Rolling Update 실습 (문제 발생)
3. ✅ 문제 원인 분석 (이미지 버전 오류)
4. ✅ Deployment 수정 및 Rolling Update 성공
5. ✅ NodePort Service 생성
6. ✅ Service 접근 문제 발생
7. ✅ 원인 분석 (kind 환경 제약사항)
8. ✅ 포트 매핑 설정 파일 생성
9. ✅ 클러스터 재생성 (포트 매핑 포함)
10. ✅ 리소스 재생성 및 접근 성공

## 다음 단계

- [ConfigMap과 Secret](../README.md) - 설정 및 민감 정보 관리
- [Ingress](../README.md) - 외부 접근 라우팅
- [StatefulSet](../README.md) - 상태가 있는 애플리케이션 관리

## 참고 자료

- [Kubernetes Deployment 문서](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Kubernetes Service 문서](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Kind 포트 매핑 문서](https://kind.sigs.k8s.io/docs/user/configuration/#extra-port-mappings)
- [Rolling Update 전략](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-update-deployment)
- [kubectl 치트시트](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
