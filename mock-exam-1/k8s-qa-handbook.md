# Kubernetes 학습 정리 - 질문과 답변 모음

## 1. Kubernetes Job - Multi-Container Pod 문제 해결

### 질문 1: Job YAML의 문제점 찾기

**초기 문제 YAML:**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: mc-pod
  namespace: mc-namespace
spec:
  template:
    spec:
      volumes:
      - name: config-vol
        emptyDir: {}
      containers:
      - name: mc-pod-1
        image: nginx:1-alpine
        env:
          - name: NODE_NAME  # ❌ 들여쓰기 오류
            valueFrom:
              fieldRef:
                fieldPath: spec.nodeName
      - name: mc-pod-2
        image: busybox:1
        command: ['sh', '-c', 'echo date >> /var/log/shared/date.log']  # ❌ 문자열 "date" 출력
        volumeMounts:
        - name: config-vol
          mountPath: /var/log/shared
      - name: mc-pod-3
        image: busybox:1
        command: ['sh', '-c', 'echo tail -f /var/log/shared/date.log']  # ❌ 문자열만 출력
        volumeMounts:
        - name: config-vol
          mountPath: /var/log/shared
      restartPolicy: OnFailure
```

**문제점:**

1. `mc-pod-1`의 `env` 리스트 항목 들여쓰기 오류 (2칸 → 6칸 필요)
2. `mc-pod-2`의 `echo date` → 문자열 "date" 출력 (실제 날짜 출력하려면 `date` 명령어 사용)
3. `mc-pod-3`의 `echo tail -f` → 문자열만 출력 (실제 tail 실행하려면 `tail -f`만 사용)

**수정된 YAML:**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: mc-pod
  namespace: mc-namespace
spec:
  template:
    spec:
      volumes:
      - name: config-vol
        emptyDir: {}
      containers:
      - name: mc-pod-1
        image: nginx:1-alpine
        env:
        - name: NODE_NAME  # ✅ 들여쓰기 수정
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
      - name: mc-pod-2
        image: busybox:1
        command: ['sh', '-c', 'date >> /var/log/shared/date.log']  # ✅ date 명령어 사용
        volumeMounts:
        - name: config-vol
          mountPath: /var/log/shared
      - name: mc-pod-3
        image: busybox:1
        command: ['sh', '-c', 'tail -f /var/log/shared/date.log']  # ✅ tail 실행
        volumeMounts:
        - name: config-vol
          mountPath: /var/log/shared
      restartPolicy: OnFailure
```

---

### 질문 2: Job의 spec.template이 immutable 에러

**에러 메시지:**

```
The Job "mc-pod" is invalid: spec.template: Invalid value: ... field is immutable
```

**원인:**

- Job의 `spec.template` 필드는 변경 불가능 (immutable)
- 기존 Job을 수정할 수 없음

**해결 방법:**

```bash
# 기존 Job 삭제 후 재생성
kubectl delete job mc-pod -n mc-namespace
kubectl apply -f 1.yaml
```

---

### 질문 3: Pod가 1/3 READY, NotReady 상태

**원인:**

- `mc-pod-2`가 Terminated (Completed) 상태
- Pod의 Ready 상태는 모든 컨테이너가 Ready: True여야 함
- 종료된 컨테이너는 Ready: False이므로 Pod 전체가 NotReady로 표시됨
- 이는 정상 동작 (Job의 특성상 컨테이너가 종료될 수 있음)

**해결:**

- `mc-pod-2`가 종료되지 않도록 무한 루프로 변경 필요

---

### 질문 4: 컨테이너가 종료되지 않게 하려면?

**문제:**

- `date >> file` 명령은 실행 후 바로 종료됨
- 컨테이너가 계속 실행되도록 해야 함

**해결:**

```yaml
command: ['sh', '-c', 'while true; do date >> /var/log/shared/date.log; sleep 1; done']
```

**설명:**

- `while true; do ... done`: 무한 루프
- `sleep 1`: 1초 대기 후 반복
- 요구사항: "every second" (매 초마다)

---

### 질문 5: Pod가 CrashLoopBackOff 상태

**원인:**

- YAML 문법 오류 (들여쓰기, 따옴표 등)
- 명령어 오류 (`while true` 구문 문제)
- 볼륨 마운트 문제

**확인 방법:**

```bash
# Pod 상세 정보
kubectl describe pod <pod-name>

# 컨테이너 로그 확인
kubectl logs <pod-name> -c <container-name>

# 이전 크래시 로그
kubectl logs <pod-name> -c <container-name> --previous
```

---

## 2. Kubernetes Service - port와 targetPort 원리

### 질문 1: Service의 port와 targetPort 차이

**원리:**

- **port**: Service가 클러스터 내에서 노출하는 포트 (외부 접근용)
- **targetPort**: Pod의 컨테이너가 실제로 리스닝하는 포트 (트래픽 전달 포트)

**예시:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: messaging-service
spec:
  selector:
    tier: msg
  ports:
    - protocol: TCP
      port: 6379      # Service 포트
      targetPort: 9376  # ❌ 잘못됨: Pod의 실제 포트와 일치해야 함
```

---

### 질문 2: Pod의 어떤 정보를 보고 targetPort를 설정해야 하나?

**확인 순서 (우선순위):**

1. **Pod YAML의 ports 섹션 확인**
   ```bash
   kubectl get pod <pod-name> -o jsonpath='{.spec.containers[0].ports}'
   ```
   - Pod describe에서 `Port: <none>` → Pod YAML에 ports가 없음

2. **컨테이너 내부 실제 포트 확인 (필수)**
   ```bash
   kubectl exec <pod-name> -- netstat -tlnp   # 또는
   kubectl exec <pod-name> -- ss -tlnp
   ```
   - 실제 리스닝 포트를 확인한 값이 `targetPort`가 되어야 함

3. **이미지의 기본 포트 사용**
   - `redis:alpine` → 6379
   - `nginx` → 80
   - `mysql` → 3306

**결론:**

- Pod describe의 `Port: <none>`만으로는 실제 포트를 알 수 없음
- 컨테이너 내부에서 실제 리스닝 포트를 확인한 뒤 그 값을 `targetPort`로 설정

---

### 질문 3: curl로 테스트했는데 "Empty reply from server"

**원인:**

- Redis는 HTTP 프로토콜이 아닌 Redis 프로토콜 사용
- `curl`은 HTTP 요청을 보내지만 Redis는 HTTP를 처리하지 않음
- 연결은 성공했지만 Redis가 HTTP를 이해하지 못해 빈 응답 반환
- **실제로는 Redis가 6379 포트에서 정상적으로 리스닝 중임을 의미**

**올바른 테스트 방법:**

```bash
# redis-cli로 테스트
kubectl run redis-client --rm -it --image=redis:alpine -- redis-cli -h messaging-service -p 6379 ping
# 응답: PONG

# Pod 내부에서 직접 테스트
kubectl exec messaging -- redis-cli ping
# 응답: PONG
```

---

### 질문 4: Service 연결 확인 방법

**확인 방법:**

```bash
# 1. Endpoints 확인 (가장 확실)
kubectl get endpoints messaging-service

# 2. Service 상세 정보
kubectl describe svc messaging-service
# Endpoints: 섹션에 Pod IP:Port 목록 표시

# 3. 실제 연결 테스트
kubectl run redis-test --rm -it --image=redis:alpine -- redis-cli -h messaging-service -p 6379 ping
```

---

## 3. Service의 Endpoints 이해

### 질문: Endpoints 2개는 무엇을 의미하나?

**의미:**

- **Endpoints = Service의 selector와 매칭되는 Pod들의 IP:Port 목록**
- Endpoints 2개 = 해당 Service에 연결된 Pod가 2개
- 각 Pod는 1개의 Endpoint를 가짐

**확인:**

```bash
kubectl get endpoints hr-web-app-service
# NAME                 ENDPOINTS                    AGE
# hr-web-app-service   172.17.0.12:8080,172.17.0.11:8080   ...
```

**원인:**

- Deployment의 `replicas: 2` 설정
- 또는 `kubectl scale` 명령으로 조정됨

---

## 4. Deployment의 replicas 이해

### 질문: Pod 2개가 자동으로 생성된 이유

**원인:**

- **Service는 Pod를 생성하지 않음**
- **Deployment가 Pod를 생성함**
- Deployment의 `replicas` 값이 Pod 개수를 결정

**확인:**

```bash
# Deployment 확인
kubectl get deploy hr-web-app

# replicas 설정 확인
kubectl get deploy hr-web-app -o jsonpath='{.spec.replicas}'

# 실제 Pod 개수 확인
kubectl get pods -l app=nginx
```

**Service vs Deployment:**

- **Service**: Pod를 생성하지 않음, 기존 Pod들을 선택해 연결만 함
- **Deployment**: `replicas: 2`로 Pod 2개 생성

**Pod 개수 조정:**

```bash
kubectl scale deploy hr-web-app --replicas=1
```

---

## 5. NodePort Service 생성

### 질문: kubectl expose 명령어 오류

**잘못된 명령어:**

```bash
kubectl expose deploy hr-web-app --port=8080 --typ=NodePort --name=hr-web-app-service --dry-run=client -o yaml > 7.yaml
# error: unknown flag: --typ
```

**문제:**

- `--typ` → 오타
- 올바른 플래그: `--type`

**수정된 명령어:**

```bash
kubectl expose deploy hr-web-app --port=8080 --type=NodePort --name=hr-web-app-service --dry-run=client -o yaml > 7.yaml
```

**생성된 YAML:**

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: nginx
  name: hr-web-app-service
spec:
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
    nodePort: 30082
  selector:
    app: nginx
  type: NodePort
```

---

## 6. Pod 수정 불가능한 필드

### 질문: Pod의 command 필드 수정 불가

**에러 메시지:**

```
The Pod "orange" is invalid: spec: Forbidden: pod updates may not change fields other than `spec.containers[*].image`, `spec.initContainers[*].image`, `spec.activeDeadlineSeconds`, `spec.tolerations` (only additions to existing tolerations), `spec.terminationGracePeriodSeconds` (allow it to be set to 1 if it was previously negative)
```

**원인:**

- Pod의 spec 필드는 대부분 변경 불가능 (immutable)
- `command` 필드는 변경할 수 없음

**변경 가능한 필드:**

- `spec.containers[*].image` (이미지만 변경 가능)
- `spec.initContainers[*].image`
- `spec.activeDeadlineSeconds`
- `spec.tolerations` (추가만 가능)
- `spec.terminationGracePeriodSeconds` (1로 설정만 가능)

**해결 방법:**

```bash
# Pod 삭제 후 재생성
kubectl delete pod orange
kubectl apply -f <yaml-file>

# 또는 replace 사용
kubectl replace -f <yaml-file>
```

---

## 7. 유용한 명령어 모음

### Pod 관련

```bash
# Pod 상세 정보
kubectl describe pod <pod-name>

# Pod 로그 확인
kubectl logs <pod-name> -c <container-name>

# 이전 크래시 로그
kubectl logs <pod-name> -c <container-name> --previous

# Pod 내부 포트 확인
kubectl exec <pod-name> -- netstat -tlnp
kubectl exec <pod-name> -- ss -tlnp

# Pod YAML 확인
kubectl get pod <pod-name> -o yaml
```

### Service 관련

```bash
# Service 목록
kubectl get svc

# Service 상세 정보
kubectl describe svc <service-name>

# Endpoints 확인
kubectl get endpoints <service-name>

# Service YAML 확인
kubectl get svc <service-name> -o yaml
```

### Deployment 관련

```bash
# Deployment 목록
kubectl get deploy

# Deployment 상세 정보
kubectl describe deploy <deployment-name>

# replicas 확인
kubectl get deploy <deployment-name> -o jsonpath='{.spec.replicas}'

# Pod 개수 조정
kubectl scale deploy <deployment-name> --replicas=2
```

### Job 관련

```bash
# Job 목록
kubectl get job

# Job 삭제
kubectl delete job <job-name> -n <namespace>

# Job의 Pod 확인
kubectl get pods -l job-name=<job-name>
```

---

## 8. 주요 개념 정리

### Job vs Deployment

- **Job**: 작업 완료 후 종료되는 Pod 생성
- **Deployment**: 지속적으로 실행되는 Pod 생성 및 관리

### Service 타입

- **ClusterIP**: 클러스터 내부에서만 접근 가능 (기본값)
- **NodePort**: 노드 IP:포트로 접근 가능
- **LoadBalancer**: 클라우드 로드밸런서와 연동
- **ExternalName**: 외부 서비스와 연결

### 볼륨 타입

- **emptyDir**: Pod 생명주기와 함께하는 임시 볼륨
- **persistentVolume**: 영구 저장 볼륨
- **configMap**: 설정 데이터 볼륨
- **secret**: 민감 정보 볼륨

### Pod 상태

- **Running**: 정상 실행 중
- **Terminated (Completed)**: 정상 종료
- **CrashLoopBackOff**: 반복적으로 크래시됨
- **NotReady**: 일부 컨테이너가 Ready 상태가 아님

### Service의 port vs targetPort

- **port**: Service가 노출하는 포트
- **targetPort**: Pod의 실제 리스닝 포트 (생략 시 port와 동일)
- **nodePort**: NodePort 타입일 때 노드에서 접근 가능한 포트

---

## 9. 문제 해결 체크리스트

### Job 생성 시

- [ ] env 리스트 항목 들여쓰기 확인 (6칸)
- [ ] 컨테이너가 종료되지 않도록 무한 루프 사용
- [ ] 볼륨 마운트 경로 확인
- [ ] restartPolicy 설정 확인

### Service 생성 시

- [ ] selector가 Pod의 label과 일치하는지 확인
- [ ] targetPort가 Pod의 실제 포트와 일치하는지 확인
- [ ] Pod YAML에 ports가 없으면 컨테이너 내부 포트 확인
- [ ] Endpoints 확인으로 Pod 연결 여부 확인

### Pod 수정 시

- [ ] Pod는 대부분의 spec 필드가 immutable
- [ ] command, args 등은 변경 불가
- [ ] 변경하려면 삭제 후 재생성 필요

### 문제 발생 시

- [ ] `kubectl describe`로 상세 정보 확인
- [ ] `kubectl logs`로 로그 확인
- [ ] `kubectl get`으로 현재 상태 확인
- [ ] YAML 문법 오류 확인 (들여쓰기, 따옴표 등)

