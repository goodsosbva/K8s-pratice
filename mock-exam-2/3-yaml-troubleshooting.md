# CKA 실습 문제 정리본

## Q.1 StorageClass 생성 및 기본 StorageClass 설정

### Task

- 이름: `local-sc`
- 기본(default) StorageClass로 설정
- Provisioner: `kubernetes.io/no-provisioner`
- VolumeBindingMode: `WaitForFirstConsumer`
- Volume 확장 허용

### Solution

**StorageClass YAML:**

```yaml
# local-sc.yaml
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
```

### Details

- `local-sc`가 `(default)`로 표시됨
- provisioner 및 옵션 값 정상 설정 확인

---

## Q.2 로그 사이드카 패턴 Deployment 생성

### Task

- Namespace: `logging-ns`
- Deployment 이름: `logging-deployment`
- Replica: 1

**main container:**

- 이름: `app-container`
- 이미지: `busybox`
- 로그 파일에 주기적으로 로그 작성

**sidecar container:**

- 이름: `log-agent`
- 이미지: `busybox`
- 로그 파일 tail

**공유 볼륨:**

- 두 컨테이너는 동일한 `emptyDir` 볼륨 사용

### Solution

**Deployment YAML:**

```yaml
# logger-deployment.yaml
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

        - name: log-agent
          image: busybox
          command:
            - sh
            - -c
            - "tail -f /var/log/app/app.log"
          volumeMounts:
            - name: log-volume
              mountPath: /var/log/app
```

**적용:**

```bash
kubectl apply -f logger-deployment.yaml
```

**로그 확인:**

```bash
kubectl logs -n logging-ns deployment/logging-deployment -c log-agent
```

### Details

- `log-agent` 컨테이너에서 "Log entry" 로그 반복 출력 확인
- 사이드카 패턴으로 로그 수집 정상 동작

---

## Q.5 사용자 john 생성 및 RBAC 설정

### Task

- 사용자: `john`
- CSR 이름: `john-developer`
- Namespace: `development`
- Pods에 대해 CRUD 권한 부여
- CSR 파일: `/root/CKA/john.csr`
- Private Key: `/root/CKA/john.key`

### Solution

**CertificateSigningRequest:**

```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: john-developer
spec:
  signerName: kubernetes.io/kube-apiserver-client
  request: <BASE64_ENCODED_CSR>
  usages:
    - digital signature
    - key encipherment
    - client auth
```

**CSR 승인:**

```bash
kubectl certificate approve john-developer
```

**Role 생성:**

```bash
kubectl create role developer \
  --resource=pods \
  --verb=create,list,get,update,delete \
  --namespace=development
```

**RoleBinding 생성:**

```bash
kubectl create rolebinding developer-role-binding \
  --role=developer \
  --user=john \
  --namespace=development
```

**권한 확인:**

```bash
kubectl auth can-i update pods --as=john --namespace=development
```

### Details

- CSR 상태: `Approved`
- User `john` → `development` namespace Pods CRUD 가능

---

## Q.6 Service / Pod DNS 해석 확인

### Task

- nginx Pod 생성
- ClusterIP Service 생성
- busybox를 사용해 DNS 및 네트워크 확인
- 결과 저장:
  - Service: `/root/CKA/nginx.svc`
  - Pod: `/root/CKA/nginx.pod`

### Solution

**Pod 및 Service 생성:**

```bash
kubectl run nginx-resolver --image=nginx
kubectl expose pod nginx-resolver \
  --name=nginx-resolver-service \
  --port=80 --target-port=80 \
  --type=ClusterIP
```

**Service DNS 확인:**

```bash
kubectl run test-nslookup \
  --image=busybox:1.28 \
  --rm -it --restart=Never \
  -- nslookup nginx-resolver-service > /root/CKA/nginx.svc
```

**Pod DNS 확인:**

```bash
# Pod IP 확인
kubectl get pod nginx-resolver -o wide

# Pod DNS 확인 (Pod IP 형식: <POD-IP>.default.pod)
kubectl run test-nslookup \
  --image=busybox:1.28 \
  --rm -it --restart=Never \
  -- nslookup <POD-IP>.default.pod > /root/CKA/nginx.pod
```

**Pod DNS 형식:**

- Pod IP: `172.17.0.12`
- DNS 형식: `172-17-0-12.default.pod.cluster.local`
- 또는: `172-17-0-12.default.pod`

### Details

- Service DNS 해석 성공
- Pod DNS 해석 성공

---

## Q.8 Horizontal Pod Autoscaler 생성

### Task

- HPA 이름: `backend-hpa`
- Namespace: `backend`
- 대상 Deployment: `backend-deployment`
- 메모리 기준 Autoscaling
- 평균 사용률: 65%
- 최소: 3 / 최대: 15

### Solution

**HPA YAML:**

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
kubectl apply -f backend-hpa.yaml
```

**확인:**

```bash
kubectl get hpa backend-hpa -n backend
```

### Details

- 메모리 사용률 65% 기준으로 자동 스케일링
- 최소 3개, 최대 15개 Pod 유지

---

## Q.9 Gateway HTTPS 설정

### Task

- Gateway 이름: `web-gateway`
- Namespace: `cka5673`
- HTTPS 443 포트
- 도메인: `kodekloud.com`
- TLS Secret: `kodekloud-tls`

### Solution

**Gateway YAML:**

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
kubectl get gateway web-gateway -n cka5673
```

### Details

- HTTPS 443 포트 리스너 설정
- TLS 인증서 참조 정상

---

## Q.10 취약한 Helm Release 제거

### Task

- 이미지: `kodekloud/webapp-color:v1`
- 해당 Helm Release 찾아 제거

### Solution

**Helm Release 찾기:**

```bash
# 모든 네임스페이스의 Helm Release 확인
helm ls -A
```

**Deployment 이미지 확인:**

```bash
# 각 Release의 Deployment 이미지 확인
kubectl get deploy -n <NAMESPACE> <DEPLOYMENT> -o json \
  | jq -r '.spec.template.spec.containers[].image'
```

**또는 직접 확인:**

```bash
# 특정 네임스페이스의 모든 Deployment 이미지 확인
kubectl get deploy -n <NAMESPACE> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.template.spec.containers[*].image}{"\n"}{end}'
```

**Helm Release 제거:**

```bash
helm uninstall <RELEASE-NAME> -n <NAMESPACE>
```

### Details

- 취약 이미지(`kodekloud/webapp-color:v1`) 포함된 Helm Release 제거 완료

---

## Q.11 NetworkPolicy 적용

### Task

- `frontend` → `backend` 허용
- `databases` → `backend` 차단
- 제공된 YAML 중 가장 restrictive 한 정책 적용
- 기존 정책 삭제 금지

### Solution

**NetworkPolicy 파일 확인:**

```bash
cat /root/net-pol-1.yaml
cat /root/net-pol-2.yaml
cat /root/net-pol-3.yaml
```

**가장 restrictive 한 정책 적용:**

```bash
kubectl apply -f /root/net-pol-3.yaml
```

**확인:**

```bash
kubectl get netpol -n backend
```

**NetworkPolicy 예시 (restrictive):**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: frontend
      ports:
        - protocol: TCP
          port: 8080
```

### Details

- `frontend` namespace만 `backend` 접근 가능
- `databases` 접근 차단
- 올바른 NetworkPolicy만 적용됨

---

## 주요 개념 정리

### StorageClass

- **기본 StorageClass 설정**: `storageclass.kubernetes.io/is-default-class: "true"` annotation 사용
- **VolumeBindingMode**: `WaitForFirstConsumer` - Pod가 생성될 때까지 볼륨 바인딩 대기
- **allowVolumeExpansion**: 볼륨 확장 허용 여부

### 사이드카 패턴

- **용도**: 로그 수집, 모니터링, 프록시 등
- **공유 볼륨**: `emptyDir` 볼륨으로 컨테이너 간 데이터 공유
- **로그 수집**: main 컨테이너가 로그 파일에 작성 → sidecar가 tail로 수집

### RBAC

- **Role**: 특정 namespace의 리소스에 대한 권한 정의
- **RoleBinding**: Role을 User/Group/ServiceAccount에 연결
- **CSR**: Certificate Signing Request - 사용자 인증서 요청

### DNS 해석

- **Service DNS**: `<service-name>.<namespace>.svc.cluster.local`
- **Pod DNS**: `<pod-ip-replaced-dashes>.<namespace>.pod.cluster.local`
- **예시**: `172-17-0-12.default.pod.cluster.local`

### HPA (Horizontal Pod Autoscaler)

- **메트릭 타입**: CPU, Memory, Custom Metrics
- **Utilization**: 평균 사용률 기준
- **minReplicas/maxReplicas**: 스케일링 범위 제한

### Gateway API

- **Gateway**: 클러스터 진입점 정의
- **HTTPS**: TLS 인증서 참조 필요
- **certificateRefs**: Secret에 저장된 TLS 인증서 참조

### NetworkPolicy

- **podSelector**: 정책이 적용될 Pod 선택
- **namespaceSelector**: 특정 namespace에서의 트래픽 허용/차단
- **restrictive**: 가장 제한적인 정책 (명시적으로 허용된 것만 통과)

---

## 유용한 명령어

### StorageClass

```bash
# StorageClass 목록 확인
kubectl get storageclass

# 기본 StorageClass 확인
kubectl get storageclass -o jsonpath='{.items[?(@.metadata.annotations.storageclass\.kubernetes\.io/is-default-class=="true")].metadata.name}'
```

### 사이드카 로그

```bash
# 특정 컨테이너 로그 확인
kubectl logs -n <namespace> deployment/<deployment-name> -c <container-name>

# 모든 컨테이너 로그 확인
kubectl logs -n <namespace> deployment/<deployment-name> --all-containers=true
```

### RBAC

```bash
# Role 확인
kubectl get role -n <namespace>

# RoleBinding 확인
kubectl get rolebinding -n <namespace>

# 권한 테스트
kubectl auth can-i <verb> <resource> --as=<user> --namespace=<namespace>
```

### DNS 테스트

```bash
# nslookup으로 DNS 확인
kubectl run test-dns --image=busybox:1.28 --rm -it --restart=Never -- nslookup <service-name>

# Pod DNS 형식 변환
# 172.17.0.12 → 172-17-0-12.default.pod
```

### HPA

```bash
# HPA 상태 확인
kubectl get hpa -n <namespace>

# HPA 상세 정보
kubectl describe hpa <hpa-name> -n <namespace>
```

### Gateway

```bash
# Gateway 확인
kubectl get gateway -n <namespace>

# Gateway 상세 정보
kubectl describe gateway <gateway-name> -n <namespace>
```

### Helm

```bash
# 모든 네임스페이스의 Release 확인
helm ls -A

# 특정 네임스페이스의 Release 확인
helm ls -n <namespace>

# Release 제거
helm uninstall <release-name> -n <namespace>
```

### NetworkPolicy

```bash
# NetworkPolicy 확인
kubectl get netpol -n <namespace>

# NetworkPolicy 상세 정보
kubectl describe netpol <netpol-name> -n <namespace>
```
