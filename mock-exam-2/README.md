# Mock Exam 2 - Kubernetes 실습 정리

## 목차

1. [StorageClass 생성](#1-storageclass-생성)
2. [Logging Deployment with Sidecar](#2-logging-deployment-with-sidecar)
3. [Ingress 생성](#3-ingress-생성)
4. [Deployment Rolling Update](#4-deployment-rolling-update)
5. [RBAC - 사용자 및 Role 생성](#5-rbac---사용자-및-role-생성)
6. [DNS Resolution 테스트](#6-dns-resolution-테스트)
7. [Static Pod 생성](#7-static-pod-생성)
8. [Horizontal Pod Autoscaler](#8-horizontal-pod-autoscaler)
9. [Gateway TLS 설정](#9-gateway-tls-설정)
10. [Helm 취약 이미지 제거](#10-helm-취약-이미지-제거)
11. [NetworkPolicy 생성](#11-networkpolicy-생성)

---

## 1. StorageClass 생성

### 문제 상황

**요구사항:**
- StorageClass 이름: `local-sc`
- Default StorageClass로 설정
- Provisioner: `kubernetes.io/no-provisioner`
- Volume Binding Mode: `WaitForFirstConsumer`
- Volume Expansion: 활성화

### 해결 방법

**YAML 파일:**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-sc
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

**적용:**
```bash
kubectl apply -f local-sc.yaml
```

**확인:**
```bash
kubectl get storageclass
# local-sc가 (default)로 표시되어야 함
```

### 핵심 개념

- **`kubernetes.io/no-provisioner`**: 동적 프로비저닝 없이 수동으로 PV 생성
- **`WaitForFirstConsumer`**: Pod가 스케줄링될 때까지 볼륨 바인딩 지연
- **`is-default-class: "true"`**: 기본 StorageClass로 설정

**인과 관계:**
```
StorageClass 생성
    ↓
annotations에 is-default-class: "true" 설정
    ↓
Kubernetes가 이를 기본 StorageClass로 인식
    ↓
PVC에서 storageClassName을 지정하지 않으면 자동으로 사용
```

---

## 2. Logging Deployment with Sidecar

### 문제 상황

**요구사항:**
- Deployment 이름: `logging-deployment`
- Namespace: `logging-ns`
- Replicas: 1
- Main Container: `app-container` (로그 생성)
- Sidecar Container: `log-agent` (로그 모니터링)

### 해결 방법

**YAML 파일:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: logging-deployment
  namespace: logging-ns
spec:
  replicas: 1
  selector:
    matchLabels:
      app: logger
  template:
    metadata:
      labels:
        app: logger
    spec:
      volumes:
        - name: log-volume
          emptyDir: {}
      initContainers:
        - name: log-agent
          image: busybox
          command:
            - sh
            - -c
            - "touch /var/log/app/app.log; tail -f /var/log/app/app.log"
          volumeMounts:
            - name: log-volume
              mountPath: /var/log/app
      containers:
        - name: app-container
          image: busybox
          command:
            - sh
            - -c
            - "while true; do echo 'Log entry' >> /var/log/app/app.log; sleep 5; done"
          volumeMounts:
            - name: log-volume
              mountPath: /var/log/app
```

**적용:**
```bash
kubectl apply -f logger-deployment.yaml
```

**확인:**
```bash
# Sidecar 로그 확인
kubectl logs -n logging-ns deployment/logging-deployment -c log-agent
```

### 주의사항

- **Init Container vs Sidecar**: 이 예제에서는 Init Container를 사용했지만, 일반적으로 `tail -f`는 Sidecar Container로 사용하는 것이 더 적절합니다.
- **공유 볼륨**: 두 컨테이너가 같은 `emptyDir` 볼륨을 마운트해야 합니다.

**인과 관계:**
```
app-container가 로그 파일에 쓰기
    ↓
공유 볼륨(emptyDir)에 저장
    ↓
log-agent가 tail -f로 파일 읽기
    ↓
log-agent의 stdout으로 출력
    ↓
kubectl logs로 확인 가능
```

---

## 3. Ingress 생성

### 문제 상황

**요구사항:**
- Ingress 이름: `webapp-ingress`
- Namespace: `ingress-ns`
- Host: `kodekloud-ingress.app`
- Path: `/` (Prefix)
- Backend Service: `webapp-svc:80`

### 해결 방법

**YAML 파일:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp-ingress
  namespace: ingress-ns
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: kodekloud-ingress.app
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: webapp-svc
            port:
              number: 80
```

**적용:**
```bash
kubectl apply -f webapp-ingress.yaml
```

**테스트:**
```bash
curl http://kodekloud-ingress.app/
```

### 핵심 개념

- **`pathType: Prefix`**: 경로가 `/`로 시작하는 모든 요청 매칭
- **`ingressClassName: nginx`**: Nginx Ingress Controller 사용
- **Host-based routing**: 호스트 이름 기반 라우팅

**인과 관계:**
```
외부 요청 (curl http://kodekloud-ingress.app/)
    ↓
Ingress Controller가 Host 헤더 확인
    ↓
kodekloud-ingress.app과 매칭되는 Ingress 규칙 찾기
    ↓
Path가 /로 시작하는지 확인 (Prefix)
    ↓
Backend Service (webapp-svc:80)로 트래픽 전달
```

---

## 4. Deployment Rolling Update

### 문제 상황

**요구사항:**
- Deployment 이름: `nginx-deploy`
- 초기 이미지: `nginx:1.16`
- 업그레이드: `nginx:1.17`
- Rolling Update 사용

### 해결 방법

**초기 생성:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.16
```

**업그레이드:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.17  # 버전 변경
```

**적용:**
```bash
kubectl apply -f nginx-deploy.yaml
```

**확인:**
```bash
kubectl get deploy nginx-deploy -w
kubectl rollout status deploy/nginx-deploy
```

### Rolling Update 동작 원리

```
기존 Pod (nginx:1.16) 실행 중
    ↓
새 Pod (nginx:1.17) 생성
    ↓
새 Pod가 Ready 상태가 될 때까지 대기
    ↓
기존 Pod 종료
    ↓
업그레이드 완료 (서비스 중단 없음)
```

**인과 관계:**
```
kubectl apply로 이미지 버전 변경
    ↓
Deployment Controller가 변경 감지
    ↓
새 ReplicaSet 생성 (nginx:1.17)
    ↓
새 Pod 생성 및 Ready 대기
    ↓
기존 Pod 종료
    ↓
Rolling Update 완료
```

---

## 5. RBAC - 사용자 및 Role 생성

### 문제 상황

**요구사항:**
- 사용자: `john`
- CSR 이름: `john-developer`
- Role: `developer` (development namespace)
- 권한: Pods에 대한 create, list, get, update, delete

### 해결 방법

**1. CSR 생성:**
```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: john-developer
spec:
  signerName: kubernetes.io/kube-apiserver-client
  request: <base64-encoded-csr>
  usages:
  - digital signature
  - key encipherment
  - client auth
```

**2. CSR 승인:**
```bash
kubectl certificate approve john-developer
```

**3. Role 생성:**
```bash
kubectl create role developer \
  --resource=pods \
  --verb=create,list,get,update,delete \
  --namespace=development
```

**4. RoleBinding 생성:**
```bash
kubectl create rolebinding developer-role-binding \
  --role=developer \
  --user=john \
  --namespace=development
```

**5. 권한 확인:**
```bash
kubectl auth can-i update pods --as=john --namespace=development
# yes
```

### 핵심 개념

- **CSR (Certificate Signing Request)**: 사용자 인증서 요청
- **Role**: 특정 네임스페이스의 리소스에 대한 권한 정의
- **RoleBinding**: Role을 사용자에게 연결
- **signerName**: Kubernetes 1.19+ 필수 필드

**인과 관계:**
```
사용자 john의 개인키로 CSR 생성
    ↓
CSR을 Kubernetes에 제출
    ↓
관리자가 CSR 승인 (kubectl certificate approve)
    ↓
인증서 발급
    ↓
Role 생성 (development namespace의 pods에 대한 권한)
    ↓
RoleBinding으로 john 사용자와 Role 연결
    ↓
john이 development namespace의 pods에 접근 가능
```

---

## 6. DNS Resolution 테스트

### 문제 상황

**요구사항:**
- Pod: `nginx-resolver` (nginx 이미지)
- Service: `nginx-resolver-service` (ClusterIP)
- DNS 해석 결과를 `/root/CKA/nginx.svc`에 저장
- Pod IP 조회 결과를 `/root/CKA/nginx.pod`에 저장

### 해결 방법

**1. Pod 및 Service 생성:**
```bash
kubectl run nginx-resolver --image=nginx
kubectl expose pod nginx-resolver \
  --name=nginx-resolver-service \
  --port=80 \
  --targetPort=80 \
  --type=ClusterIP
```

**2. Service DNS 조회:**
```bash
kubectl run test-nslookup \
  --image=busybox:1.28 \
  --rm -it \
  --restart=Never \
  -- nslookup nginx-resolver-service > /root/CKA/nginx.svc
```

**3. Pod IP 조회:**
```bash
# Pod IP 확인 (예: 10.244.1.5)
kubectl get pod nginx-resolver -o wide

# IP를 하이픈으로 변환 (10.244.1.5 → 10-244-1-5)
# Pod DNS 형식: <POD-IP>.<NAMESPACE>.pod.cluster.local
kubectl run test-nslookup \
  --image=busybox:1.28 \
  --rm -it \
  --restart=Never \
  -- nslookup 10-244-1-5.default.pod > /root/CKA/nginx.pod
```

### DNS 형식

- **Service DNS**: `<service-name>.<namespace>.svc.cluster.local`
- **Pod DNS**: `<pod-ip-with-hyphens>.<namespace>.pod.cluster.local`

**인과 관계:**
```
Service 생성
    ↓
CoreDNS가 Service DNS 레코드 생성
    ↓
형식: <service-name>.<namespace>.svc.cluster.local
    ↓
nslookup으로 해석 가능

Pod 생성
    ↓
CoreDNS가 Pod DNS 레코드 생성
    ↓
형식: <pod-ip-with-hyphens>.<namespace>.pod.cluster.local
    ↓
nslookup으로 해석 가능
```

---

## 7. Static Pod 생성

### 문제 상황

**요구사항:**
- Static Pod 이름: `nginx-critical`
- 노드: `node01`
- 이미지: `nginx`
- 자동 재시작 설정

### 해결 방법

**1. Pod YAML 생성:**
```bash
kubectl run nginx-critical \
  --image=nginx \
  --dry-run=client -o yaml > static.yaml
```

**2. node01로 파일 전송:**
```bash
scp static.yaml node01:/root/
```

**3. node01에 SSH 접속:**
```bash
ssh node01
```

**4. Static Pod 디렉토리 생성:**
```bash
mkdir -p /etc/kubernetes/manifests
```

**5. kubelet 설정 확인/수정:**
```bash
vi /var/lib/kubelet/config.yaml
# staticPodPath: /etc/kubernetes/manifests 확인
```

**6. YAML 파일 복사:**
```bash
cp /root/static.yaml /etc/kubernetes/manifests/
```

**7. 확인:**
```bash
# controlplane에서
kubectl get pods
# nginx-critical-node01 표시됨
```

### Static Pod 특징

- **kubelet이 직접 관리**: API Server를 거치지 않음
- **자동 재시작**: Pod 실패 시 자동으로 재생성
- **노드별 관리**: 각 노드의 kubelet이 독립적으로 관리

**인과 관계:**
```
/etc/kubernetes/manifests/ 디렉토리에 YAML 파일 배치
    ↓
kubelet이 디렉토리 모니터링
    ↓
YAML 파일 발견 시 Pod 생성
    ↓
Pod 실패 시 자동으로 재생성
    ↓
API Server에 미러 Pod로 등록 (읽기 전용)
```

---

## 8. Horizontal Pod Autoscaler

### 문제 상황

**요구사항:**
- HPA 이름: `backend-hpa`
- Namespace: `backend`
- Target: `backend-deployment`
- Metric: Memory utilization 65%
- Min Replicas: 3
- Max Replicas: 15

### 해결 방법

**YAML 파일:**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
  namespace: backend
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend-deployment
  minReplicas: 3
  maxReplicas: 15
  metrics:
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 65
```

**적용:**
```bash
kubectl create -f webapp-hpa.yaml
```

**확인:**
```bash
kubectl get hpa backend-hpa -n backend
```

### 주의사항

- **Utilization vs AverageValue**:
  - `Utilization`: 사용률 (0-100 숫자)
  - `AverageValue`: 절대값 (예: "100Mi", "1Gi")
- **Deployment에 resources 설정 필요**: HPA가 작동하려면 Deployment의 Pod에 `resources.requests` 설정이 필요합니다.

**인과 관계:**
```
HPA 생성 및 Deployment 타겟 지정
    ↓
HPA가 metrics-server에서 메모리 사용률 수집
    ↓
평균 메모리 사용률이 65%를 초과
    ↓
Pod 수 증가 (최대 15개까지)
    ↓
평균 메모리 사용률이 65% 미만으로 감소
    ↓
Pod 수 감소 (최소 3개까지)
```

### 잘못된 설정 예시

```yaml
# ❌ 잘못된 설정
target:
  type: AverageValue
  averageValue: 65%  # 퍼센트는 사용 불가
```

**올바른 설정:**
```yaml
# ✅ 올바른 설정
target:
  type: Utilization
  averageUtilization: 65  # 숫자만 사용 (65% 의미)
```

---

## 9. Gateway TLS 설정

### 문제 상황

**요구사항:**
- Gateway 이름: `web-gateway`
- Namespace: `cka5673`
- Protocol: HTTPS
- Port: 443
- Hostname: `kodekloud.com`
- TLS Secret: `kodekloud-tls`

### 해결 방법

**YAML 파일:**
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: web-gateway
  namespace: cka5673
spec:
  gatewayClassName: kodekloud
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      hostname: kodekloud.com
      tls:
        certificateRefs:
          - name: kodekloud-tls
```

**적용:**
```bash
kubectl apply -f web-gateway.yaml
```

**확인:**
```bash
kubectl get gateway web-gateway -n cka5673 -o yaml
```

### 변경 사항

- **Protocol**: HTTP → HTTPS
- **Port**: 80 → 443
- **TLS 설정 추가**: `certificateRefs`로 Secret 참조

**인과 관계:**
```
기존 Gateway (HTTP, Port 80)
    ↓
HTTPS로 변경 (Protocol: HTTPS, Port: 443)
    ↓
TLS 인증서 Secret 참조 추가
    ↓
Hostname 지정 (kodekloud.com)
    ↓
HTTPS 트래픽 처리 가능
```

---

## 10. Helm 취약 이미지 제거

### 문제 상황

**요구사항:**
- 취약 이미지: `kodekloud/webapp-color:v1` 사용하는 Helm release 찾기
- 해당 release 제거

### 해결 방법

**1. 모든 네임스페이스의 Helm release 확인:**
```bash
helm ls -A
```

**2. 각 Deployment의 이미지 확인:**
```bash
# 각 네임스페이스와 Deployment에 대해
kubectl get deploy -n <NAMESPACE> <DEPLOYMENT-NAME> -o json | \
  jq -r '.spec.template.spec.containers[].image'
```

**3. 취약 이미지 사용하는 release 제거:**
```bash
helm uninstall <RELEASE-NAME> -n <NAMESPACE>
```

### 예시 스크립트

```bash
# 모든 release 확인
helm ls -A

# 각 release의 이미지 확인
for ns in $(helm ls -A -o json | jq -r '.[].namespace' | sort -u); do
  for release in $(helm ls -n $ns -o json | jq -r '.[].name'); do
    echo "Checking $release in $ns"
    # Deployment 이름 확인 후 이미지 확인
  done
done

# 취약 이미지 발견 시
helm uninstall <RELEASE-NAME> -n <NAMESPACE>
```

**인과 관계:**
```
helm ls -A로 모든 release 확인
    ↓
각 release의 Deployment 이미지 확인
    ↓
kodekloud/webapp-color:v1 이미지 사용하는 release 발견
    ↓
helm uninstall로 release 제거
    ↓
취약 이미지 제거 완료
```

---

## 11. NetworkPolicy 생성

### 문제 상황

**요구사항:**
- `frontend` 네임스페이스에서 `backend` 네임스페이스로 트래픽 허용
- `databases` 네임스페이스에서 `backend` 네임스페이스로 트래픽 차단
- 가장 제한적인 정책 적용

### 해결 방법

**1. 제공된 정책 파일 확인:**
```bash
cat /root/net-pol-1.yaml
cat /root/net-pol-2.yaml
cat /root/net-pol-3.yaml
```

**2. 올바른 정책 분석:**
- **net-pol-1.yaml**: 너무 광범위 (특정 라벨만으로 허용)
- **net-pol-2.yaml**: 잘못됨 (frontend와 databases 모두 허용)
- **net-pol-3.yaml**: 올바름 (frontend만 허용)

**3. 올바른 정책 적용:**
```bash
kubectl apply -f /root/net-pol-3.yaml
```

**4. 확인:**
```bash
kubectl get netpol -n backend
# net-policy-3만 표시되어야 함
```

### NetworkPolicy 예시 (net-pol-3.yaml)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: net-policy-3
  namespace: backend
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: frontend
```

### 핵심 개념

- **`podSelector: {}`**: 모든 Pod에 적용
- **`namespaceSelector`**: 특정 네임스페이스에서 오는 트래픽만 허용
- **가장 제한적인 정책**: 필요한 트래픽만 허용하고 나머지는 차단

**인과 관계:**
```
NetworkPolicy 생성 (backend namespace)
    ↓
podSelector: {} (모든 Pod에 적용)
    ↓
ingress 규칙: frontend 네임스페이스만 허용
    ↓
frontend → backend: 허용 ✅
databases → backend: 차단 ❌
```

---

## 주요 명령어 요약

### StorageClass
```bash
kubectl get storageclass
kubectl describe storageclass local-sc
```

### Deployment & Sidecar
```bash
kubectl get deploy -n logging-ns
kubectl logs -n logging-ns deployment/logging-deployment -c log-agent
```

### Ingress
```bash
kubectl get ingress -n ingress-ns
curl http://kodekloud-ingress.app/
```

### RBAC
```bash
kubectl get csr
kubectl certificate approve <csr-name>
kubectl get role,rolebinding -n development
kubectl auth can-i <verb> <resource> --as=<user> -n <namespace>
```

### DNS
```bash
kubectl run test --image=busybox:1.28 --rm -it --restart=Never -- nslookup <service-name>
```

### Static Pod
```bash
# controlplane에서
kubectl get pods
# node01에서
ls /etc/kubernetes/manifests/
```

### HPA
```bash
kubectl get hpa -n backend
kubectl describe hpa backend-hpa -n backend
```

### Gateway
```bash
kubectl get gateway -n cka5673
kubectl describe gateway web-gateway -n cka5673
```

### Helm
```bash
helm ls -A
helm uninstall <release-name> -n <namespace>
```

### NetworkPolicy
```bash
kubectl get netpol -n backend
kubectl describe netpol <name> -n backend
```

---

## 핵심 교훈

1. **StorageClass**: Default StorageClass는 `annotations`로 설정
2. **Sidecar Pattern**: 공유 볼륨을 통해 컨테이너 간 통신
3. **Ingress**: Host-based routing과 path-based routing 구분
4. **Rolling Update**: `kubectl apply`로 자동 Rolling Update 수행
5. **RBAC**: Role과 RoleBinding으로 네임스페이스별 권한 관리
6. **DNS**: Service와 Pod의 DNS 형식 이해
7. **Static Pod**: kubelet이 직접 관리하는 Pod
8. **HPA**: Utilization vs AverageValue 구분 중요
9. **Gateway**: TLS 설정은 `certificateRefs` 사용
10. **Helm**: `helm ls -A`로 모든 네임스페이스 확인
11. **NetworkPolicy**: 가장 제한적인 정책 적용 원칙

---

## 문제 해결 체크리스트

### StorageClass
- [ ] `is-default-class: "true"` annotation 설정
- [ ] `provisioner: kubernetes.io/no-provisioner` 확인
- [ ] `volumeBindingMode: WaitForFirstConsumer` 확인
- [ ] `allowVolumeExpansion: true` 확인

### Sidecar
- [ ] 공유 볼륨(emptyDir) 설정
- [ ] 두 컨테이너 모두 볼륨 마운트
- [ ] 로그 확인: `kubectl logs -c log-agent`

### Ingress
- [ ] `pathType: Prefix` 설정
- [ ] `hostname` 설정
- [ ] `backend.service` 설정
- [ ] `ingressClassName` 설정

### HPA
- [ ] `type: Utilization` (퍼센트 사용 시)
- [ ] `averageUtilization: 65` (숫자만, % 없음)
- [ ] Deployment에 `resources.requests` 설정 확인

### NetworkPolicy
- [ ] `namespaceSelector`로 정확한 네임스페이스 지정
- [ ] 가장 제한적인 정책 선택
- [ ] 기존 정책 확인 후 적용

