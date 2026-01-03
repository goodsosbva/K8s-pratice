# Stateless 애플리케이션 보안 설정 실습

## 개요

이 실습은 Kubernetes에서 Stateless 애플리케이션을 안전하고 견고하게 만드는 방법을 학습합니다. 문제가 있는 Deployment 설정을 분석하고, Health Check, 보안 설정, 불필요한 리소스 제거 등을 통해 프로덕션 수준의 안전한 Deployment로 개선합니다.

## 학습 목표

1. **Health Check 설정 이해**

   - Readiness Probe와 Liveness Probe의 차이와 용도
   - 올바른 포트와 경로 설정
   - Health Check 실패 시 동작 이해

2. **보안 설정 학습**

   - Security Context 설정
   - Non-root 사용자 실행
   - 불필요한 권한 제거

3. **리소스 최적화**

   - 불필요한 사이드카 컨테이너 제거
   - 리소스 제한 설정
   - 효율적인 컨테이너 구성

4. **리소스 제한 문제 진단**

   - OOMKilled (Exit Code 137) 이해
   - CrashLoopBackOff 상태 분석
   - Pod의 lastState 확인 방법
   - 리소스 사용량 모니터링

5. **문제 진단 및 해결**
   - `kubectl describe`로 문제 분석
   - `kubectl edit`로 실시간 수정
   - YAML 파일 수정 및 재적용
   - `kubectl get`의 jsonpath 활용

## 사전 준비

### 1. 이전 실습 완료

이 실습을 진행하기 전에 다음 실습을 완료해야 합니다:

- [make-cluster 실행 정리](../make-cluster/README.md) - Kind 클러스터 생성
- [make-the-pod 실행 정리](../make-the-pod/README.md) - Pod 생성 실습
- [try-debugging 실행 정리](../try-debugging/README.md) - Pod 디버깅 실습
- [break-and-fix 실행 정리](../break-and-fix/README.md) - Deployment와 Service 실습

### 2. 환경 설정

```bash
# 프로젝트 루트에서 환경 설정 적용
source .bashrc

# kubectl alias 설정 (선택사항)
alias k=kubectl
```

### 3. hello-server 애플리케이션 확인

hello-server는 포트 8080에서 동작하며, `/health` 엔드포인트는 제공하지 않습니다. 따라서 Health Check는 TCP 소켓 연결 또는 루트 경로(`/`)를 사용해야 합니다.

## 실습 파일

### deployment-destruction.yaml (문제가 있는 파일)

이 파일에는 여러 문제가 있습니다:

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
          image: blux2/hello-server:1.6
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /health # ❌ 존재하지 않는 경로
              port: 8081 # ❌ 존재하지 않는 포트
            initialDelaySeconds: 5
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health # ❌ 존재하지 않는 경로
              port: 8080 # ✅ 올바른 포트
            initialDelaySeconds: 10
            periodSeconds: 5
        - name: busybox # ❌ 불필요한 사이드카 컨테이너
          image: busybox:1.36.1
          command:
            - sleep
            - "9999"
```

**문제점:**

1. **Readiness Probe 오류**

   - 포트 8081을 참조하지만 애플리케이션은 8080에서만 동작
   - `/health` 경로가 존재하지 않음

2. **Liveness Probe 오류**

   - `/health` 경로가 존재하지 않음
   - 루트 경로(`/`)를 사용하거나 TCP 소켓 체크를 사용해야 함

3. **불필요한 사이드카 컨테이너**

   - `busybox` 컨테이너는 디버깅 목적으로만 사용되며 프로덕션에 불필요
   - 리소스 낭비 및 보안 위험 증가

4. **보안 설정 부재**
   - Security Context가 설정되지 않음
   - Non-root 사용자 설정 없음
   - 리소스 제한 없음

### deployment-resource-handson.yaml (리소스 제한 예시)

이 파일은 리소스 제한 설정 예시를 보여줍니다:

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
          image: blux2/hello-server:1.6
          ports:
            - containerPort: 8080
          resources:
            requests:
              memory: "5Gi"
              cpu: "10m"
            limits:
              memory: "5Gi"
              cpu: "10m"
          readinessProbe:
            httpGet:
              path: /health # ⚠️ 여전히 존재하지 않는 경로
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health # ⚠️ 여전히 존재하지 않는 경로
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
```

**참고사항:**

- 리소스 제한 설정은 올바르지만, Health Check 경로는 여전히 수정이 필요합니다.

### deployment-memory-leak.yaml (메모리 제한 실습 파일)

이 파일은 메모리 제한이 낮게 설정된 Deployment입니다. OOMKilled 문제를 재현하기 위한 실습 파일입니다:

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
          image: blux2/hello-server:1.7
          ports:
            - containerPort: 8080
          resources:
            requests:
              memory: "100Mi"
              cpu: "10m"
            limits:
              memory: "100Mi" # ⚠️ 메모리 제한이 낮음 (OOMKilled 발생 가능)
              cpu: "10m"
```

**실습 목적:**

- 메모리 제한 초과 시 Pod 재시작 동작 확인
- Exit Code 137 (OOMKilled) 이해
- CrashLoopBackOff 상태 확인
- 리소스 제한 조정 방법 학습

## 실습 단계

### 1. 문제가 있는 Deployment 생성

```bash
# make-stateless-application-secure 디렉토리로 이동
cd make-stateless-application-secure

# 문제가 있는 Deployment 생성
kubectl apply -f deployment-destruction.yaml
```

**출력 예시:**

```
deployment.apps/hello-server created
```

### 2. Pod 상태 확인

```bash
# Pod 목록 확인
kubectl get pods -l app=hello-server
```

**예상 출력:**

```
NAME                            READY   STATUS    RESTARTS   AGE
hello-server-xxxxx-xxxxx        0/1     Running   0          30s
hello-server-xxxxx-xxxxx        0/1     Running   0          30s
hello-server-xxxxx-xxxxx        0/1     Running   0          30s
```

**관찰 사항:**

- `READY` 상태가 `0/1`로 표시됨 (Readiness Probe 실패)
- Pod는 실행 중이지만 트래픽을 받을 준비가 되지 않음

### 3. Pod 상세 정보 확인

```bash
# Pod 상세 정보 확인
kubectl describe pod -l app=hello-server
```

**중요한 정보 확인:**

1. **Readiness Probe 실패:**

   ```
   Readiness:     http-get http://:8081/health delay=5s timeout=1s period=5s #success=1 #failure=3
   Liveness:      http-get http://:8080/health delay=10s timeout=1s period=5s #success=1 #failure=3
   ```

2. **Events 섹션:**

   ```
   Warning  Unhealthy  Readiness probe failed: Get "http://10.244.0.5:8081/health": dial tcp 10.244.0.5:8081: connect: connection refused
   Warning  Unhealthy  Liveness probe failed: Get "http://10.244.0.5:8080/health": http: no Host in request URL
   ```

3. **컨테이너 정보:**
   - `hello-server` 컨테이너와 `busybox` 컨테이너가 모두 실행 중

### 4. 문제 분석

**발견된 문제:**

1. **Readiness Probe 실패**

   - 포트 8081에 연결 시도하지만 애플리케이션은 8080에서만 동작
   - 연결 거부(connection refused) 오류 발생

2. **Liveness Probe 실패**

   - `/health` 경로가 존재하지 않음
   - HTTP 404 또는 경로 오류 발생

3. **불필요한 리소스**
   - `busybox` 컨테이너가 실행 중이지만 실제로 사용되지 않음

### 5. 문제 해결 방법 1: kubectl edit 사용

```bash
# Deployment 편집
kubectl edit deploy hello-server
```

**수정할 내용:**

1. **Readiness Probe 수정:**

   ```yaml
   readinessProbe:
     httpGet:
       path: / # /health → / 로 변경
       port: 8080 # 8081 → 8080으로 변경
     initialDelaySeconds: 5
     periodSeconds: 5
   ```

2. **Liveness Probe 수정:**

   ```yaml
   livenessProbe:
     httpGet:
       path: / # /health → / 로 변경
       port: 8080
     initialDelaySeconds: 10
     periodSeconds: 5
   ```

3. **busybox 컨테이너 제거:**

   - `busybox` 컨테이너 전체 섹션 삭제

4. **보안 설정 추가 (선택사항):**

   ```yaml
   securityContext:
     runAsNonRoot: true
     runAsUser: 1000
     allowPrivilegeEscalation: false
     readOnlyRootFilesystem: false
   ```

5. **리소스 제한 추가:**
   ```yaml
   resources:
     requests:
       memory: "64Mi"
       cpu: "100m"
     limits:
       memory: "128Mi"
       cpu: "200m"
   ```

**편집기 종료:**

- vi: `:wq` (저장 후 종료)
- nano: `Ctrl+X` → `Y` → `Enter`

### 6. 문제 해결 방법 2: YAML 파일 수정 후 재적용

**올바른 Deployment 설정 파일 생성:**

`deployment-secure.yaml` 파일 생성:

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
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        allowPrivilegeEscalation: false
      containers:
        - name: hello-server
          image: blux2/hello-server:1.6
          ports:
            - containerPort: 8080
          resources:
            requests:
              memory: "64Mi"
              cpu: "100m"
            limits:
              memory: "128Mi"
              cpu: "200m"
          readinessProbe:
            httpGet:
              path: /
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
            timeoutSeconds: 1
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
            timeoutSeconds: 1
            failureThreshold: 3
```

**적용:**

```bash
# 기존 Deployment 삭제
kubectl delete deploy hello-server

# 수정된 Deployment 적용
kubectl apply -f deployment-secure.yaml
```

### 7. 수정 후 상태 확인

```bash
# Pod 상태 확인
kubectl get pods -l app=hello-server
```

**예상 출력:**

```
NAME                            READY   STATUS    RESTARTS   AGE
hello-server-xxxxx-xxxxx        1/1     Running   0          2m
hello-server-xxxxx-xxxxx        1/1     Running   0          2m
hello-server-xxxxx-xxxxx        1/1     Running   0          2m
```

**개선 사항:**

- `READY` 상태가 `1/1`로 변경됨 (Readiness Probe 성공)
- `busybox` 컨테이너가 제거되어 리소스 사용량 감소
- 보안 설정이 적용됨

### 8. Pod 상세 정보 재확인

```bash
# Pod 상세 정보 확인
kubectl describe pod -l app=hello-server
```

**확인할 내용:**

1. **Readiness Probe 성공:**

   ```
   Readiness:     http-get http://:8080/ delay=5s timeout=1s period=5s #success=1 #failure=3
   ```

2. **컨테이너 개수:**

   - `hello-server` 컨테이너만 존재 (busybox 제거됨)

3. **보안 설정:**

   ```
   Security Context:
     Run As User:  1000
     Run As Non Root:  true
   ```

4. **리소스 제한:**
   ```
   Limits:
     cpu:     200m
     memory:  128Mi
   Requests:
     cpu:     100m
     memory:  64Mi
   ```

## 리소스 제한 실습 (메모리 리크 시나리오)

### 실습 목표

리소스 제한이 너무 낮게 설정된 경우 발생하는 문제를 확인하고, Pod의 재시작 상태를 분석하는 방법을 학습합니다.

### 1. 메모리 제한이 낮은 Deployment 생성

```bash
# make-stateless-application-secure 디렉토리로 이동
cd make-stateless-application-secure

# 메모리 제한이 낮은 Deployment 생성 (100Mi)
kubectl apply -f deployment-memory-leak.yaml
```

**deployment-memory-leak.yaml 내용:**

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
          image: blux2/hello-server:1.7
          ports:
            - containerPort: 8080
          resources:
            requests:
              memory: "100Mi"
              cpu: "10m"
            limits:
              memory: "100Mi" # ⚠️ 메모리 제한이 낮음
              cpu: "10m"
```

### 2. Pod 상태 모니터링

```bash
# Pod 상태를 실시간으로 모니터링
kubectl get pods -w
```

**초기 상태:**

```
NAME                            READY   STATUS    RESTARTS   AGE
hello-server-856d5c6c7b-lkwlh   1/1     Running   1          2m19s
hello-server-856d5c6c7b-qvzsg   1/1     Running   0          2m19s
hello-server-856d5c6c7b-vbb6g   1/1     Running   0          2m19s
```

**메모리 제한 초과 시:**

```
NAME                            READY   STATUS              RESTARTS   AGE
hello-server-856d5c6c7b-lkwlh   0/1     Error               1          10m
hello-server-856d5c6c7b-qvzsg   1/1     Running             0          10m
hello-server-856d5c6c7b-vbb6g   1/1     Running             0          10m
hello-server-856d5c6c7b-lkwlh   0/1     CrashLoopBackOff    1 (16s ago)   10m
hello-server-856d5c6c7b-lkwlh   1/1     Running             2 (20s ago)   10m
```

**관찰 사항:**

- Pod가 `Error` 상태로 변경됨
- `CrashLoopBackOff` 상태로 전환됨
- `RESTARTS` 카운트가 증가함
- 일정 시간 후 자동으로 재시작됨

### 3. Pod의 종료 상태 확인

```bash
# Pod의 lastState 확인 (종료된 컨테이너 상태)
kubectl get pod hello-server-856d5c6c7b-lkwlh -o=jsonpath="{.status.containerStatuses[0].lastState}"
```

**출력 예시:**

```json
{
  "terminated": {
    "containerID": "containerd://69177dd18b65a4cfc71e6f35469c3513d5d98ecfc183fb26dc1cbb2e80025bf0",
    "exitCode": 137,
    "finishedAt": "2026-01-02T10:59:42Z",
    "reason": "Error",
    "startedAt": "2026-01-02T10:51:05Z"
  }
}
```

**중요한 정보:**

- **exitCode: 137** - OOMKilled (Out Of Memory Killed)를 의미
  - 128 + 9 (SIGKILL) = 137
  - Kubernetes가 메모리 제한을 초과한 컨테이너를 강제 종료함
- **reason: "Error"** - 컨테이너가 오류로 종료됨
- **finishedAt / startedAt** - 컨테이너의 실행 시간 확인 가능

### 4. Pod 상세 정보 확인

```bash
# Pod 상세 정보 확인
kubectl describe pod hello-server-856d5c6c7b-lkwlh
```

**확인할 내용:**

1. **리소스 제한:**

   ```
   Limits:
     cpu:     10m
     memory:  100Mi
   Requests:
     cpu:     10m
     memory:  100Mi
   ```

2. **Events 섹션:**

   ```
   Warning  Failed   Error: container hello-server is OOMKilled (memory limit exceeded)
   ```

3. **상태 정보:**
   ```
   State:          Running
   Last State:     Terminated
     Reason:       Error
     Exit Code:    137
   ```

### 5. 리소스 사용량 확인 (metrics-server 필요)

```bash
# Pod의 리소스 사용량 확인
kubectl top pod hello-server-856d5c6c7b-lkwlh

# 모든 Pod의 리소스 사용량 확인
kubectl top pods -l app=hello-server
```

**출력 예시:**

```
NAME                            CPU(cores)   MEMORY(bytes)
hello-server-856d5c6c7b-lkwlh   5m           150Mi  # ⚠️ 제한(100Mi) 초과
hello-server-856d5c6c7b-qvzsg    3m           80Mi
hello-server-856d5c6c7b-vbb6g   4m           85Mi
```

### 6. 문제 해결: 리소스 제한 증가

```bash
# Deployment 편집
kubectl edit deploy hello-server
```

**수정할 내용:**

```yaml
resources:
  requests:
    memory: "100Mi"
    cpu: "10m"
  limits:
    memory: "200Mi" # 100Mi → 200Mi로 증가
    cpu: "100m" # 10m → 100m으로 증가
```

**또는 YAML 파일 수정 후 재적용:**

```bash
# deployment-memory-leak.yaml 파일 수정 후
kubectl apply -f deployment-memory-leak.yaml
```

### 7. Rolling Update 확인

```bash
# Rolling Update 진행 상황 확인
kubectl get pods -w
```

**출력 예시:**

```
NAME                            READY   STATUS              RESTARTS   AGE
hello-server-58654c5c9f-5ccjk   0/1     ContainerCreating   0          5s
hello-server-856d5c6c7b-lkwlh   1/1     Running             2 (82s ago)   11m
hello-server-856d5c6c7b-qvzsg    1/1     Running             0             11m
hello-server-856d5c6c7b-vbb6g   1/1     Running             0             11m
```

**관찰 사항:**

- 새로운 ReplicaSet이 생성됨 (`hello-server-58654c5c9f-*`)
- 새로운 Pod가 점진적으로 생성됨
- 기존 Pod는 새 Pod가 준비될 때까지 유지됨

### 8. 최종 상태 확인

```bash
# Pod 상태 확인
kubectl get pods -l app=hello-server

# 서비스 테스트 (포트 포워딩 또는 Service 사용)
kubectl port-forward deployment/hello-server 8080:8080
```

**다른 터미널에서:**

```bash
curl localhost:8080
```

**예상 출력:**

```
Hello, world! Let's learn Kubernetes!
```

### 9. 주요 학습 포인트

1. **Exit Code 137의 의미**

   - OOMKilled를 나타냄
   - 메모리 제한 초과 시 Kubernetes가 컨테이너를 강제 종료

2. **CrashLoopBackOff 동작**

   - Pod가 반복적으로 실패하면 지수 백오프로 재시작 간격 증가
   - 문제 해결 후 자동으로 정상 상태로 복구

3. **리소스 제한 설정의 중요성**

   - 너무 낮으면: OOMKilled 발생
   - 너무 높으면: 리소스 낭비 및 노드 스케줄링 문제

4. **Rolling Update의 안전성**
   - 리소스 제한 변경 시에도 서비스 중단 없이 업데이트 가능
   - 새 Pod가 준비될 때까지 기존 Pod 유지

### 10. 유용한 명령어

```bash
# Pod의 종료 상태 확인
kubectl get pod <pod-name> -o=jsonpath="{.status.containerStatuses[0].lastState}"

# Exit Code만 확인
kubectl get pod <pod-name> -o=jsonpath="{.status.containerStatuses[0].lastState.terminated.exitCode}"

# 종료 이유 확인
kubectl get pod <pod-name> -o=jsonpath="{.status.containerStatuses[0].lastState.terminated.reason}"

# Pod 이벤트 확인
kubectl describe pod <pod-name> | grep -A 20 "Events"

# 리소스 사용량 확인 (metrics-server 필요)
kubectl top pod <pod-name>

# Deployment 롤아웃 상태 확인
kubectl rollout status deploy hello-server

# Deployment 롤아웃 히스토리 확인
kubectl rollout history deploy hello-server
```

## 주요 개념 설명

### Readiness Probe vs Liveness Probe

**Readiness Probe (준비 상태 확인):**

- Pod가 트래픽을 받을 준비가 되었는지 확인
- 실패 시: Pod는 Service의 Endpoint에서 제거됨
- 용도: 애플리케이션 초기화 완료 확인

**Liveness Probe (생존 상태 확인):**

- Pod가 정상적으로 동작하는지 확인
- 실패 시: Pod가 재시작됨
- 용도: 애플리케이션 장애 감지 및 자동 복구

### Health Check 타입

**HTTP GET:**

```yaml
httpGet:
  path: /
  port: 8080
```

**TCP 소켓:**

```yaml
tcpSocket:
  port: 8080
```

**명령어 실행:**

```yaml
exec:
  command:
    - cat
    - /tmp/healthy
```

### Security Context

**Pod 레벨 설정:**

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 1000
```

**Container 레벨 설정:**

```yaml
containers:
  - name: hello-server
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: false
      capabilities:
        drop:
          - ALL
```

### 리소스 제한

**Requests (요청량):**

- Kubernetes 스케줄러가 노드 선택 시 고려하는 최소 리소스
- 보장되는 리소스

**Limits (제한량):**

- 컨테이너가 사용할 수 있는 최대 리소스
- 초과 시 컨테이너가 제한됨 (OOMKilled 가능)

## 주요 명령어

### Deployment 관리

```bash
# Deployment 생성
kubectl apply -f deployment-destruction.yaml

# Deployment 상태 확인
kubectl get deploy hello-server

# Deployment 상세 정보
kubectl describe deploy hello-server

# Deployment 편집
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

# Pod 로그 확인
kubectl logs <pod-name> -c hello-server

# Pod 내부 접속
kubectl exec -it <pod-name> -c hello-server -- /bin/sh
```

### Health Check 확인

```bash
# Readiness Probe 상태 확인
kubectl get pods -l app=hello-server -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.conditions[?(@.type=="Ready")].status}{"\n"}{end}'

# Liveness Probe 이벤트 확인
kubectl describe pod <pod-name> | grep -A 5 "Liveness"
```

### 리소스 사용량 확인

```bash
# Pod 리소스 사용량 확인 (metrics-server 필요)
kubectl top pods -l app=hello-server

# 노드 리소스 사용량 확인
kubectl top nodes
```

## 트러블슈팅

### 1. Readiness Probe 계속 실패

**증상:**

```
READY   STATUS
0/1     Running
```

**원인:**

- 잘못된 포트 번호
- 존재하지 않는 경로
- 애플리케이션이 아직 시작되지 않음

**해결:**

```bash
# Pod 로그 확인
kubectl logs <pod-name> -c hello-server

# Pod 내부에서 포트 확인
kubectl exec <pod-name> -c hello-server -- netstat -tlnp

# Health Check 경로 테스트
kubectl exec <pod-name> -c hello-server -- wget -qO- http://localhost:8080/
```

### 2. Liveness Probe로 인한 Pod 재시작 반복

**증상:**

```
RESTARTS   AGE
5          2m
```

**원인:**

- Liveness Probe 경로가 잘못됨
- 애플리케이션이 응답하지 않음
- initialDelaySeconds가 너무 짧음

**해결:**

```bash
# 재시작 이벤트 확인
kubectl describe pod <pod-name> | grep -A 10 "Events"

# Liveness Probe 설정 확인
kubectl get pod <pod-name> -o yaml | grep -A 10 livenessProbe

# initialDelaySeconds 증가
kubectl edit deploy hello-server
```

### 3. Security Context 오류

**증상:**

```
Error: container has runAsNonRoot and image will run as root
```

**원인:**

- 이미지가 root 사용자로 실행되도록 설정됨
- runAsUser가 설정되지 않음

**해결:**

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000 # 명시적으로 사용자 ID 지정
```

### 4. 리소스 제한으로 인한 Pod 실패

**증상:**

```
STATUS              RESTARTS
CrashLoopBackOff    5
Error               1
```

또는

```
STATUS
OOMKilled
```

**원인:**

- 메모리 제한이 너무 낮음
- 애플리케이션이 limits를 초과함
- Exit Code 137 (OOMKilled) 발생

**진단:**

```bash
# Pod의 종료 상태 확인
kubectl get pod <pod-name> -o=jsonpath="{.status.containerStatuses[0].lastState}"

# Exit Code 확인 (137 = OOMKilled)
kubectl get pod <pod-name> -o=jsonpath="{.status.containerStatuses[0].lastState.terminated.exitCode}"

# 종료 이유 확인
kubectl get pod <pod-name> -o=jsonpath="{.status.containerStatuses[0].lastState.terminated.reason}"

# 리소스 사용량 확인 (metrics-server 필요)
kubectl top pod <pod-name>

# Pod 이벤트 확인
kubectl describe pod <pod-name> | grep -A 10 "Events"
```

**해결:**

```bash
# 리소스 제한 증가
kubectl edit deploy hello-server

# 또는 YAML 파일 수정 후 재적용
kubectl apply -f deployment-memory-leak.yaml
```

**수정 예시:**

```yaml
resources:
  requests:
    memory: "100Mi"
    cpu: "10m"
  limits:
    memory: "200Mi" # 100Mi → 200Mi로 증가
    cpu: "100m" # 10m → 100m으로 증가
```

**예방:**

- 애플리케이션의 실제 메모리 사용량을 모니터링하여 적절한 제한 설정
- requests와 limits의 차이를 적절히 설정 (버퍼 제공)
- 메모리 사용량이 예상보다 높다면 애플리케이션 최적화 고려

## 실습 단계 요약

### 기본 실습 (Health Check 및 보안 설정)

1. ✅ 문제가 있는 Deployment 생성
2. ✅ Pod 상태 확인 (Readiness Probe 실패 확인)
3. ✅ Pod 상세 정보 분석 (문제 원인 파악)
4. ✅ Health Check 설정 수정 (포트 및 경로 수정)
5. ✅ 불필요한 사이드카 컨테이너 제거
6. ✅ 보안 설정 추가 (Security Context)
7. ✅ 리소스 제한 설정
8. ✅ 수정된 Deployment 적용 및 검증

### 리소스 제한 실습 (메모리 리크 시나리오)

1. ✅ 메모리 제한이 낮은 Deployment 생성
2. ✅ Pod 상태 모니터링 (CrashLoopBackOff 확인)
3. ✅ Pod의 종료 상태 확인 (exitCode 137 - OOMKilled)
4. ✅ 리소스 사용량 확인 및 분석
5. ✅ 리소스 제한 증가 (Deployment 수정)
6. ✅ Rolling Update 진행 상황 확인
7. ✅ 최종 상태 검증 및 서비스 테스트

## 다음 단계

- [ConfigMap과 Secret](../README.md) - 설정 및 민감 정보 관리
- [Ingress](../README.md) - 외부 접근 라우팅
- [StatefulSet](../README.md) - 상태가 있는 애플리케이션 관리

## 참고 자료

- [Kubernetes Health Checks 문서](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Kubernetes Security Context 문서](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
- [Kubernetes Resource Management 문서](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Kubernetes Deployment 문서](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
