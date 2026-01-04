# Mock Exam 1 - Kubernetes 실습 정리

## 목차

1. [Pod 생성 및 에러 해결](#pod-생성-및-에러-해결)
2. [CRD 조회 및 필터링](#crd-조회-및-필터링)
3. [Service 생성 및 연결 문제 해결](#service-생성-및-연결-문제-해결)
4. [Port 설정 이해 (port, targetPort, nodePort)](#port-설정-이해)
5. [Persistent Volume 생성](#persistent-volume-생성)
6. [Horizontal Pod Autoscaler 생성](#horizontal-pod-autoscaler-생성)
7. [Helm Chart 업그레이드](#helm-chart-업그레이드)
8. [주요 에러 해결 과정](#주요-에러-해결-과정)

---

## Pod 생성 및 에러 해결

### mc-pod 생성 시 발생한 에러

**요구사항:**

- Name: mc-pod
- Namespace: mc-namespace
- 3개의 컨테이너 (nginx, busybox 2개)
- 공유 볼륨 사용

**에러 1: fieldPath 대소문자 오류**

```bash
kubectl apply -f 1.yaml
# Error: field label not supported: spec.nodename
```

**원인:**

- `spec.nodename` (소문자)로 작성함
- Kubernetes Downward API는 대소문자를 엄격히 구분함

**해결:**

```yaml
env:
  - name: NODE_NAME
    valueFrom:
      fieldRef:
        fieldPath: spec.nodeName # N이 대문자!
```

**에러 2: 컨테이너 이름 형식 위반**

```bash
# Error: Invalid value: "busybox:1": a lowercase RFC 1123 label must consist of...
```

**원인:**

- 컨테이너 이름에 콜론(`:`) 사용 불가
- RFC 1123 규칙: 소문자, 숫자, 하이픈(`-`)만 허용

**해결:**

```yaml
containers:
  - image: busybox:1
    name: mc-pod-2 # busybox:1 → mc-pod-2로 변경
```

**에러 3: ImagePullBackOff**

```bash
kubectl get pods -n mc-namespace
# NAME     READY   STATUS             RESTARTS   AGE
# mc-pod   2/3     ImagePullBackOff   0          11s
```

**원인:**

- `busybox:1` 태그가 공식 저장소에 없을 가능성
- 이미지 태그 오류 또는 네트워크 문제

**해결 방법:**

```yaml
# busybox:1 대신 다음 중 하나 사용
- image: busybox # latest 태그 사용
- image: busybox:1.36 # 정확한 버전 명시
```

**최종 YAML:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mc-pod
  namespace: mc-namespace
spec:
  containers:
    - image: nginx:1-alpine
      name: mc-pod-1
      env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName # 대소문자 정확히!
      volumeMounts:
        - mountPath: /var/log/shared
          name: cache-volume
    - image: busybox:1
      name: mc-pod-2 # 콜론 제거!
      command:
        - "sh"
        - "-c"
        - "while true; do date >> /var/log/shared/date.log; sleep 1; done"
      volumeMounts:
        - mountPath: /var/log/shared
          name: cache-volume
    - image: busybox:1
      name: mc-pod-3
      command:
        - "sh"
        - "-c"
        - "tail -f /var/log/shared/date.log"
      volumeMounts:
        - mountPath: /var/log/shared
          name: cache-volume
  volumes:
    - name: cache-volume
      emptyDir: {}
```

---

## CRD 조회 및 필터링

### VPA 관련 CRD 이름 추출

**요구사항:**

- VerticalPodAutoscaler 관련 CRD 찾기
- 이름만 추출하여 `/root/vpa-crds.txt`에 저장

**방법 1: jsonpath + grep (추천)**

```bash
kubectl get crds -o jsonpath='{.items[*].metadata.name}' | tr ' ' '\n' | grep -i verticalpodautoscaler > /root/vpa-crds.txt
```

**설명:**

- `-o jsonpath='{.items[*].metadata.name}'`: 모든 CRD 이름을 공백으로 구분하여 한 줄로 출력
- `tr ' ' '\n'`: 공백을 줄바꿈으로 변환 (grep이 한 줄씩 읽을 수 있게)
- `grep -i verticalpodautoscaler`: 대소문자 구분 없이 필터링
- `> /root/vpa-crds.txt`: 파일로 저장

**방법 2: awk 사용 (간단)**

```bash
kubectl get crds | grep -i verticalpodautoscaler | awk '{print $1}' > /root/vpa-crds.txt
```

**방법 3: custom-columns 사용**

```bash
kubectl get crds -o custom-columns=NAME:.metadata.name --no-headers | grep -i verticalpodautoscaler > /root/vpa-crds.txt
```

**확인:**

```bash
cat /root/vpa-crds.txt
# 예상 출력:
# verticalpodautoscalers.autoscaling.k8s.io
# verticalpodautoscalercheckpoints.autoscaling.k8s.io
```

---

## Service 생성 및 연결 문제 해결

### 1. messaging-service (ClusterIP) 생성

**요구사항:**

- Name: messaging-service
- Port: 6379
- Type: ClusterIP
- Selector: 올바른 라벨 사용

**문제 발생:**

```bash
curl -v 172.20.20.113:6379
# Connection refused 발생
```

**원인 분석:**

1. **라벨 불일치**: 서비스의 selector와 파드의 labels가 일치하지 않음

   - 서비스 selector: `app: messaging`
   - 실제 파드 labels: `tier=msg`

2. **엔드포인트 확인:**

```bash
kubectl get endpoints messaging-service
# ENDPOINTS: <none> (연결된 파드 없음)
```

**해결 방법:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: messaging-service
spec:
  type: ClusterIP
  selector:
    tier: msg # 파드의 실제 라벨과 일치시킴
  ports:
    - port: 6379
      targetPort: 6379
      protocol: TCP
```

**확인:**

```bash
# 1. 서비스 IP 확인
kubectl get svc messaging-service
# NAME                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
# messaging-service   ClusterIP   172.20.20.113   <none>        6379/TCP   4s

# 2. 엔드포인트 확인
kubectl get endpoints messaging-service
# ENDPOINTS: 172.17.0.8:6379 (정상 연결)

# 3. curl로 연결 확인 (CLUSTER-IP 사용)
curl -v 172.20.20.113:6379
# Connected to 172.20.20.113 port 6379 (성공)
```

**Service 연결 확인 방법:**

1. **서비스 IP 확인:**

   ```bash
   kubectl get svc <service-name>
   # CLUSTER-IP 컬럼의 IP 주소 확인
   ```

2. **엔드포인트 확인:**

   ```bash
   kubectl get endpoints <service-name>
   # ENDPOINTS에 파드 IP가 있으면 정상 연결
   ```

3. **curl로 연결 테스트:**
   ```bash
   # 위에서 확인한 CLUSTER-IP 사용
   curl -v <CLUSTER-IP>:<port>
   ```

---

### 2. hr-web-app-service (NodePort) 생성

**요구사항:**

- Name: hr-web-app-service
- Type: NodePort
- Port: 8080
- NodePort: 30082
- Endpoints: 2개

**문제 발생 과정:**

#### 문제 1: Connection Refused

```bash
curl 172.17.0.10:30082
# Connection refused
```

**원인:** targetPort를 30082로 설정함 (잘못된 설정)

- NodePort(30082)는 외부 노드 포트이지, 파드 내부 포트가 아님

#### 문제 2: targetPort 80으로 설정해도 실패

```bash
curl 172.17.0.10:80
# Connection refused
```

**원인 분석:**

- `kubectl describe deploy`에서 Port: 80/TCP로 표시됨
- 하지만 실제 컨테이너는 8080 포트에서 실행 중

**실제 포트 확인:**

```bash
# 파드 IP로 직접 접속 테스트
curl -v 172.17.0.10:8080
# Connected to 172.17.0.10 port 8080 (성공)
# HTML 응답 수신

# 로그 확인
kubectl logs hr-web-app-f59fc74cc-rftdl
# * Running on http://0.0.0.0:8080/ (Press CTRL+C to quit)
```

**핵심 발견:**

- Deployment의 `containerPort: 80`은 단순 표기용 (메타데이터)
- 실제 애플리케이션(Flask)은 8080 포트에서 실행 중
- 이미지(`kodekloud/webapp-color`)가 내부적으로 8080을 사용

**최종 해결:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hr-web-app-service
spec:
  type: NodePort
  selector:
    app: nginx # Deployment의 라벨
  ports:
    - protocol: TCP
      port: 8080 # 서비스 포트
      targetPort: 8080 # 파드의 실제 포트 (중요!)
      nodePort: 30082 # 외부 접속 포트
```

**확인:**

```bash
# 엔드포인트 확인
kubectl get endpoints hr-web-app-service
# ENDPOINTS: 172.17.0.10:8080,172.17.0.11:8080 (2개 정상)

# NodePort로 접속 확인
curl localhost:30082
# HTML 응답 수신 (성공)
```

---

## Port 설정 이해

### port, targetPort, nodePort 차이

| 구분           | 설명                   | 위치                | 용도                                                     |
| -------------- | ---------------------- | ------------------- | -------------------------------------------------------- |
| **port**       | 서비스가 노출하는 포트 | Service (ClusterIP) | 클러스터 내부의 다른 파드들이 이 서비스를 호출할 때 사용 |
| **targetPort** | 파드 내부의 실제 포트  | Pod (Container)     | 서비스가 요청을 최종적으로 전달할 포트 번호              |
| **nodePort**   | 외부 노드 포트         | Node (Host)         | 클러스터 외부에서 접속하기 위해 여는 포트 (30000-32767)  |

### 통신 흐름 (NodePort 예시)

```
사용자 (curl localhost:30082)
  ↓
노드의 30082 포트 (nodePort)
  ↓
서비스의 8080 포트 (port)
  ↓
파드의 8080 포트 (targetPort)
  ↓
컨테이너 애플리케이션 응답
```

### 주의사항

1. **containerPort는 표기용**: Deployment의 `containerPort`는 메타데이터일 뿐, 실제 포트를 강제하지 않음
2. **실제 포트 확인 방법:**

   ```bash
   # 방법 1: 로그 확인
   kubectl logs <pod-name>
   # * Running on http://0.0.0.0:8080/

   # 방법 2: 파드 IP로 직접 테스트
   curl <pod-ip>:8080

   # 방법 3: 컨테이너 내부 확인
   kubectl exec <pod-name> -- netstat -tuln
   ```

---

## Persistent Volume 생성

**요구사항:**

- Volume name: pv-analytics
- Storage: 100Mi
- Access mode: ReadWriteMany
- Host path: /pv/data-analytics

**YAML:**

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-analytics
spec:
  capacity:
    storage: 100Mi
  accessModes:
    - ReadWriteMany
  hostPath:
    path: /pv/data-analytics
```

**hostPath 설명:**

- **의미**: 파드가 떠 있는 노드(Host)의 실제 디렉토리 경로
- **용도**: 파드가 죽어도 데이터가 노드의 로컬 디렉토리에 영구 저장됨
- **주의**: 멀티 노드 환경에서는 파드가 다른 노드로 이동하면 데이터를 찾을 수 없음

**확인:**

```bash
kubectl get pv pv-analytics
kubectl describe pv pv-analytics
```

---

## Horizontal Pod Autoscaler 생성

**요구사항:**

- Name: webapp-hpa
- Target: kkapp-deploy deployment
- CPU utilization: 50% 유지
- Scale down stabilization: 300초

**YAML:**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: webapp-hpa
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: kkapp-deploy
  minReplicas: 1
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300 # 5분 대기 후 천천히 축소
```

**설명:**

- **averageUtilization: 50**: 파드들의 평균 CPU 사용량이 50%를 넘으면 파드 증가
- **stabilizationWindowSeconds: 300**: 부하가 줄어도 300초(5분) 동안 지켜본 후 파드 축소 (급격한 변동 방지)

**확인:**

```bash
kubectl get hpa webapp-hpa
# TARGETS: 0%/50% (CPU 사용률 / 목표치)
```

**주의사항:**

- HPA가 작동하려면 Deployment의 파드에 `resources.requests.cpu` 설정이 필요함
- 설정이 없으면 TARGETS에 `<unknown>/50%` 표시됨

---

## 주요 에러 해결 과정

### 1. Service 연결 실패 (Connection Refused)

**증상:**

```bash
curl -v <service-ip>:<port>
# Connection refused
```

**원인:**

1. Selector와 Pod Labels 불일치
2. targetPort가 실제 컨테이너 포트와 불일치

**해결 순서:**

```bash
# 1. 엔드포인트 확인
kubectl get endpoints <service-name>
# <none>이면 selector 문제

# 2. 파드 라벨 확인
kubectl get pods --show-labels

# 3. 서비스 selector 확인
kubectl describe svc <service-name> | grep Selector

# 4. 실제 포트 확인
kubectl logs <pod-name>
# 또는
curl <pod-ip>:<port>
```

### 2. Port 불일치 문제

**증상:**

- `kubectl describe`에는 Port: 80으로 표시
- 실제로는 8080에서 실행 중

**원인:**

- Deployment의 `containerPort`는 메타데이터일 뿐
- 실제 포트는 이미지 내부 애플리케이션이 결정

**해결:**

- 로그나 직접 테스트로 실제 포트 확인
- Service의 `targetPort`를 실제 포트로 설정

### 3. YAML 구조 오류

**주요 에러:**

- `unknown field "spec.spec"`: spec 중복
- `unknown field "spec.ports[0].type"`: type 위치 오류 (ports 안이 아닌 spec 바로 아래)

**해결:**

- YAML 구조 정확히 확인
- `type`은 `spec` 바로 아래, `ports`는 그 안에

---

## 체크리스트

### Service 생성 시 확인사항

- [ ] Selector가 파드의 Labels와 정확히 일치하는가?
- [ ] targetPort가 실제 컨테이너 포트와 일치하는가?
- [ ] port, targetPort, nodePort 설정이 올바른가?
- [ ] 엔드포인트에 파드 IP가 등록되었는가? (`kubectl get endpoints`)

### Port 확인 방법

1. **로그 확인**: `kubectl logs <pod-name>`
2. **직접 테스트**: `curl <pod-ip>:<port>`
3. **컨테이너 내부**: `kubectl exec <pod-name> -- netstat -tuln`

### Service 연결 확인

```bash
# 1. 서비스 IP 확인 (CLUSTER-IP 컬럼 확인)
kubectl get svc <service-name>
# 예: kubectl get svc messaging-service
# CLUSTER-IP: 172.20.20.113

# 2. 엔드포인트 확인
kubectl get endpoints <service-name>
# ENDPOINTS에 파드 IP가 있으면 정상

# 3. 서비스 상세 정보
kubectl describe svc <service-name>
# Selector와 Endpoints 확인

# 4. curl로 연결 테스트
# 위에서 확인한 CLUSTER-IP 사용
curl -v <CLUSTER-IP>:<port>
# 예: curl -v 172.20.20.113:6379

# 또는 NodePort인 경우
curl localhost:<nodePort>
```

---

## Helm Chart 업그레이드

### 문제 상황

**요구사항:**

- 동료가 `kk-mock1`이라는 nginx Helm chart를 `kk-ns` 네임스페이스에 배포함
- 새로운 버전(18.1.15)이 Helm 저장소에 푸시됨
- Helm 저장소를 업데이트하고 chart를 18.1.15 버전으로 업그레이드해야 함

### 왜 업그레이드가 필요한가?

**인과 관계:**

```
새 버전(18.1.15)이 저장소에 푸시됨
    ↓
기존 배포는 이전 버전(18.1.0)을 사용 중
    ↓
버그 수정, 보안 패치, 새 기능이 반영되지 않음
    ↓
업그레이드 필요
```

### 단계별 해결 과정

#### 1단계: 현재 배포 상태 확인

**명령어:**

```bash
helm list -n kk-ns
```

**출력:**

```
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART           APP VERSION
kk-mock1        kk-ns           1               2026-01-03 08:54:51.777058012 +0000 UTC deployed        nginx-18.1.0    1.27.0
```

**왜 필요한가?**

- 현재 배포된 chart 이름과 버전 확인
- 업그레이드 대상 릴리스 이름 확인 (`kk-mock1`)
- 현재 버전(18.1.0)과 목표 버전(18.1.15) 비교

**인과 관계:**

```
현재 상태를 모르면 업그레이드 불가
    ↓
helm list로 현재 배포 정보 확인
    ↓
업그레이드 대상과 현재 버전 파악
```

#### 2단계: Helm 저장소 업데이트

**명령어:**

```bash
helm repo update
```

**출력:**

```
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "kk-mock1" chart repository
Update Complete. ⎈Happy Helming!⎈
```

**왜 필요한가?**

**Helm 저장소 동작 원리:**

```
원격 Helm 저장소 (최신 버전 정보)
    ↓
로컬 Helm 캐시 (성능을 위해 저장)
    ↓
helm repo update 실행
    ↓
로컬 캐시가 최신 정보로 갱신됨
```

**인과 관계:**

```
새 버전이 저장소에 푸시됨
    ↓
로컬 Helm 캐시는 여전히 이전 정보를 가지고 있음
    ↓
helm repo update 실행
    ↓
로컬 캐시가 최신 정보로 갱신됨
    ↓
이제 18.1.15 버전을 인식할 수 있음
```

**설명:**

- Helm은 성능을 위해 저장소 정보를 로컬에 캐시함
- 저장소에 새 버전이 추가되어도 자동으로 갱신되지 않음
- `helm repo update`를 실행해야 최신 정보를 가져올 수 있음
- 이 단계를 건너뛰면 `--version 18.1.15`를 지정해도 "version not found" 오류 발생

#### 3단계: 사용 가능한 버전 확인

**명령어:**

```bash
helm search repo nginx --versions | grep 18.1.15
```

**출력:**

```
kk-mock1/nginx                          18.1.15         1.27.1          NGINX Open Source is a web server that can be a...
```

**왜 필요한가?**

**인과 관계:**

```
helm repo update로 최신 정보 가져옴
    ↓
이제 사용 가능한 버전 목록을 확인할 수 있음
    ↓
18.1.15 버전이 존재하는지 확인
    ↓
정확한 chart 이름 확인 (kk-mock1/nginx)
```

**설명:**

- 저장소마다 chart 이름이 다를 수 있음
- `helm search repo`로 정확한 chart 경로 확인 필요
- 버전이 존재하는지 확인 후 업그레이드 진행
- 여기서 확인한 chart 경로(`kk-mock1/nginx`)를 업그레이드 명령어에 사용

#### 4단계: 특정 버전으로 업그레이드

**명령어:**

```bash
helm upgrade kk-mock1 kk-mock1/nginx --version 18.1.15 -n kk-ns
```

**출력:**

```
Release "kk-mock1" has been upgraded. Happy Helming!
NAME: kk-mock1
LAST DEPLOYED: Sat Jan  3 09:17:17 2026
NAMESPACE: kk-ns
STATUS: deployed
REVISION: 2
TEST SUITE: None
NOTES:
CHART NAME: nginx
CHART VERSION: 18.1.15
APP VERSION: 1.27.1
```

**왜 이렇게 해야 하는가?**

**Helm 업그레이드 동작 원리:**

```
helm upgrade 명령어 실행
    ↓
Helm이 현재 배포된 리소스 상태 확인
    ↓
새 버전(18.1.15)의 chart 템플릿 렌더링
    ↓
현재 상태와 새 상태 비교 (diff)
    ↓
변경된 부분만 업데이트 (Rolling Update)
    ↓
기존 Pod는 새 Pod가 준비될 때까지 유지
    ↓
새 Pod가 준비되면 기존 Pod 종료
    ↓
서비스 중단 없이 업그레이드 완료
```

**`--version` 옵션이 필요한 이유:**

- `--version` 없이 실행하면 최신 버전으로 업그레이드됨
- 요구사항은 정확히 18.1.15로 업그레이드
- 특정 버전을 명시해야 정확한 버전으로 업그레이드됨

**인과 관계:**

```
helm repo update (최신 정보 가져옴)
    ↓
18.1.15 버전이 사용 가능해짐
    ↓
helm upgrade --version 18.1.15 (특정 버전 지정)
    ↓
Helm이 18.1.15 버전의 chart를 다운로드
    ↓
기존 배포와 비교하여 변경사항 적용
    ↓
Rolling Update로 서비스 중단 없이 업그레이드
```

**명령어 구성 요소:**

- `kk-mock1`: 업그레이드할 릴리스 이름
- `kk-mock1/nginx`: chart 경로 (저장소명/chart명)
- `--version 18.1.15`: 업그레이드할 정확한 버전
- `-n kk-ns`: 네임스페이스 지정

#### 5단계: 업그레이드 확인

**명령어:**

```bash
helm list -n kk-ns
```

**출력:**

```
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART           APP VERSION
kk-mock1        kk-ns           2               2026-01-03 09:17:17.159044339 +0000 UTC deployed        nginx-18.1.15   1.27.1
```

**확인 포인트:**

- **REVISION**: 1 → 2로 증가 (업그레이드 성공)
- **CHART**: nginx-18.1.0 → nginx-18.1.15 (버전 업그레이드)
- **APP VERSION**: 1.27.0 → 1.27.1 (애플리케이션 버전도 업데이트)
- **STATUS**: deployed (정상 배포됨)

### 전체 흐름도

```
[문제 상황]
동료가 nginx chart 배포 (18.1.0 버전)
    ↓
새 버전(18.1.15)이 저장소에 푸시됨
    ↓
[1단계: 상태 확인]
helm list -n kk-ns
→ 현재 배포 상태 확인 (18.1.0 버전)
    ↓
[2단계: 저장소 업데이트]
helm repo update
→ 로컬 캐시를 최신 정보로 갱신
→ 18.1.15 버전을 인식할 수 있게 됨
    ↓
[3단계: 버전 확인]
helm search repo nginx --versions | grep 18.1.15
→ 18.1.15 버전 존재 확인
→ 정확한 chart 이름 확인 (kk-mock1/nginx)
    ↓
[4단계: 업그레이드]
helm upgrade kk-mock1 kk-mock1/nginx --version 18.1.15 -n kk-ns
→ 18.1.15 버전의 chart 다운로드
→ 기존 리소스와 비교
→ Rolling Update 실행
→ 서비스 중단 없이 업그레이드 완료
    ↓
[5단계: 확인]
helm list -n kk-ns
→ 업그레이드 성공 확인 (REVISION: 2, CHART: nginx-18.1.15)
```

### 각 단계를 건너뛰면?

#### 2단계(helm repo update)를 건너뛰면?

- 로컬 캐시에 18.1.15 버전 정보가 없음
- `--version 18.1.15`를 지정해도 "version not found" 오류 발생
- **결과**: 업그레이드 실패

#### 3단계(버전 확인)를 건너뛰면?

- 잘못된 chart 이름으로 업그레이드 시도 가능
- 존재하지 않는 버전 지정 가능
- **결과**: 오류 발생 후 다시 확인해야 함

#### 4단계에서 `--version`을 생략하면?

- 최신 버전으로 업그레이드됨 (18.1.15가 아닐 수 있음)
- 요구사항(18.1.15)과 다를 수 있음
- **결과**: 요구사항 미충족

### 핵심 개념

#### Helm 저장소 캐시

- Helm은 성능을 위해 저장소 정보를 로컬에 캐시함
- 새 버전이 추가되어도 자동 갱신되지 않음
- `helm repo update`로 수동 갱신 필요

#### Helm 업그레이드 동작

- 기존 리소스와 새 리소스를 비교하여 변경사항만 적용
- Rolling Update 방식으로 서비스 중단 없이 업그레이드
- REVISION 번호가 증가하여 업그레이드 이력 관리

#### 버전 지정의 중요성

- `--version` 옵션으로 정확한 버전 지정 가능
- 생략 시 최신 버전으로 업그레이드 (요구사항과 다를 수 있음)
- 프로덕션 환경에서는 특정 버전 명시 권장

### 유용한 명령어

```bash
# 현재 배포 상태 확인
helm list -n <namespace>

# 저장소 업데이트
helm repo update

# 사용 가능한 버전 확인
helm search repo <chart-name> --versions

# 특정 버전으로 업그레이드
helm upgrade <release-name> <chart-path> --version <version> -n <namespace>

# 업그레이드 상태 확인
helm status <release-name> -n <namespace>

# 업그레이드 이력 확인
helm history <release-name> -n <namespace>

# 업그레이드 롤백 (필요시)
helm rollback <release-name> <revision-number> -n <namespace>
```

---

## 핵심 교훈

1. **라벨 일치가 생명**: Service의 selector와 Pod의 labels가 토씨 하나 안 틀리고 일치해야 함
2. **실제 포트 확인 필수**: describe의 Port는 참고용, 실제 포트는 로그나 테스트로 확인
3. **엔드포인트 확인**: `kubectl get endpoints`로 연결 상태를 가장 빠르게 확인 가능
4. **containerPort는 표기용**: 실제 포트를 강제하지 않으므로 신뢰하지 말 것
5. **Helm 저장소 업데이트 필수**: 새 버전을 사용하려면 반드시 `helm repo update` 실행
6. **버전 명시의 중요성**: `--version` 옵션으로 정확한 버전 지정하여 요구사항 충족
