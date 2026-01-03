# Kubernetes 구조와 아키텍처 이해 실습

## 개요

이 실습은 Kubernetes의 멀티 노드 클러스터 구조를 이해하고, 애플리케이션을 배포한 후 모니터링 시스템(Prometheus + Grafana)을 구축하는 전체 과정을 학습합니다. 실습을 통해 Kubernetes의 노드 구조, 네트워킹, 서비스 디스커버리, 그리고 모니터링 아키텍처를 종합적으로 이해할 수 있습니다.

## 학습 목표

1. **멀티 노드 클러스터 구조 이해**

   - Control Plane과 Worker 노드의 역할
   - 노드 간 네트워크 통신
   - NodePort Service의 동작 원리

2. **애플리케이션 배포 및 서비스 노출**

   - Deployment를 통한 애플리케이션 배포
   - NodePort Service를 통한 외부 접근
   - 네임스페이스 분리

3. **모니터링 시스템 구축**

   - Prometheus를 통한 메트릭 수집
   - Grafana를 통한 시각화
   - 커스텀 애플리케이션 메트릭 수집

4. **네트워크 및 포트 매핑 이해**
   - Kind 클러스터의 포트 매핑 동작
   - NodePort와 포트 매핑의 관계
   - 클러스터 내부/외부 접근 차이

## 실습 흐름 및 인과 관계

### 1단계: 멀티 노드 클러스터 생성

**목적**: 실제 프로덕션 환경과 유사한 멀티 노드 구조를 구성하여 노드 간 통신과 서비스 분산을 이해합니다.

**왜 필요한가?**

- 단일 노드 클러스터는 제한적이며, 실제 환경에서는 여러 노드에 Pod가 분산 배치됩니다
- NodePort Service는 모든 노드에서 접근 가능해야 하므로 멀티 노드 환경에서 테스트해야 합니다
- 노드 간 네트워크 통신과 서비스 디스커버리를 이해하기 위해 필요합니다

**실행**:

```bash
# 멀티 노드 클러스터 생성
kind create cluster --name multinode-nodeport --config multinode-nodeport.yaml --image=kindest/node:v1.29.0

# 노드 확인
kubectl get nodes -o wide

# 각 노드의 IP 확인
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.addresses[?(@.type=="InternalIP")].address}{"\n"}{end}'
```

**결과 확인**:

- Control Plane 노드 1개
- Worker 노드 2개
- 각 노드의 InternalIP 확인

### 2단계: 네임스페이스 생성

**목적**: 리소스를 논리적으로 분리하여 관리합니다.

**왜 필요한가?**

- 개발 환경과 모니터링 환경을 분리하여 관리
- 리소스 격리 및 권한 관리
- 실제 프로덕션 환경의 네임스페이스 전략 학습

**실행**:

```bash
# develop 네임스페이스 생성
kubectl apply -f namespace.yaml

# 네임스페이스 확인
kubectl get namespace develop
```

### 3단계: 애플리케이션 배포

**목적**: Prometheus 메트릭을 노출하는 애플리케이션을 배포합니다.

**왜 이렇게 구성했는가?**

#### 3-1. 애플리케이션 특징 (main.go)

```go
http.Handle("/metrics", promhttp.Handler())  // Prometheus 메트릭 엔드포인트
http.HandleFunc("/healthz", ...)             // Health Check 엔드포인트
```

**설계 이유**:

- `/metrics` 엔드포인트: Prometheus가 메트릭을 수집할 수 있도록 표준 엔드포인트 제공
- `/healthz` 엔드포인트: Kubernetes의 Health Check에 사용
- Go의 `prometheus/client_golang` 라이브러리 사용으로 자동으로 Go 런타임 메트릭 노출

#### 3-2. Deployment 설정 (hello-server.yaml)

**주요 설정**:

- `replicas: 3`: 고가용성을 위한 다중 인스턴스
- `readinessProbe` / `livenessProbe`: `/healthz` 경로 사용
- `resources`: 메모리/CPU 제한 설정

**설계 이유**:

- 다중 Pod로 장애 복구 및 로드 분산
- Health Check로 Pod의 준비 상태와 생존 상태 확인
- 리소스 제한으로 노드 리소스 보호

**실행**:

```bash
# Deployment 및 Service 생성
kubectl apply -f hello-server.yaml

# Pod 상태 확인
kubectl get pods -n develop -o wide

# Service 확인
kubectl get svc -n develop
```

**결과 확인**:

- 3개의 Pod가 서로 다른 노드에 분산 배치됨
- NodePort Service가 생성되어 외부 접근 가능

### 4단계: NodePort Service 접근 테스트

**목적**: NodePort Service가 모든 노드에서 접근 가능한지 확인합니다.

**왜 이렇게 동작하는가?**

#### 4-1. NodePort Service 동작 원리

```
클라이언트 요청
    ↓
어떤 노드의 IP:NodePort로 접근
    ↓
해당 노드의 kube-proxy가 요청을 받음
    ↓
Service의 Endpoints로 라우팅
    ↓
실제 Pod로 트래픽 전달
```

**핵심 개념**:

- NodePort Service는 **모든 노드**에서 동일한 포트로 리스닝합니다
- 어떤 노드 IP로 접근해도 동일한 서비스에 접근 가능합니다
- kube-proxy가 노드 간 트래픽을 프록시합니다

#### 4-2. Kind 환경의 포트 매핑

**문제 상황**:

```bash
curl 172.18.0.4:30599  # 실패
```

**원인 분석**:

- `multinode-nodeport.yaml`에서 포트 매핑이 첫 번째 worker에만 설정됨
- `listenAddress: "127.0.0.1"`로 설정되어 localhost에만 바인딩됨
- Docker 네트워크 IP(172.18.0.x)는 호스트에서 직접 접근 불가

**해결 방법**:

```bash
# localhost로 접근 (포트 매핑이 127.0.0.1에 바인딩됨)
curl localhost:30599

# 또는 클러스터 내부에서 접근 (모든 노드 IP로 접근 가능)
kubectl run test --image=curlimages/curl --rm -it -- curl 172.18.0.4:30599
```

**학습 포인트**:

- Kubernetes 클러스터 내부에서는 모든 노드 IP로 접근 가능
- 호스트에서 접근하려면 포트 매핑이 필요하며, `listenAddress` 설정에 따라 접근 범위가 결정됨

### 5단계: Prometheus 설치 및 설정

**목적**: 메트릭 수집 시스템을 구축하여 애플리케이션과 클러스터 상태를 모니터링합니다.

**왜 Prometheus를 사용하는가?**

- Kubernetes 생태계의 표준 모니터링 도구
- Pull 기반 메트릭 수집으로 효율적
- ServiceMonitor를 통한 자동 타겟 발견
- 강력한 쿼리 언어(PromQL) 제공

#### 5-1. Helm을 통한 설치

**왜 Helm을 사용하는가?**

- 복잡한 Prometheus Operator 설정을 간소화
- values.yaml을 통한 설정 관리
- 업그레이드 및 롤백 용이

**실행**:

```bash
# Helm repo 추가
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# monitoring 네임스페이스 생성
kubectl create namespace monitoring

# Prometheus Stack 설치
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack -n monitoring
```

#### 5-2. 커스텀 Scrape Config 설정

**목적**: hello-server 애플리케이션의 메트릭을 수집합니다.

**설정 내용 (values.yaml)**:

```yaml
prometheus:
  prometheusSpec:
    additionalScrapeConfigs:
      - job_name: hello-server
        scrape_interval: 10s
        static_configs:
          - targets:
              - hello-server.develop.svc.cluster.local:8080
```

**왜 이렇게 설정했는가?**

- `job_name: hello-server`: Prometheus에서 이 타겟을 식별하는 이름
- `scrape_interval: 10s`: 10초마다 메트릭 수집 (빠른 테스트를 위해)
- `hello-server.develop.svc.cluster.local`: Kubernetes DNS를 통한 서비스 디스커버리
  - `hello-server`: Service 이름
  - `develop`: 네임스페이스
  - `svc.cluster.local`: Kubernetes 클러스터 도메인

**설계 이유**:

- Kubernetes의 내부 DNS를 활용하여 Pod IP 변경에 영향받지 않음
- 네임스페이스 분리를 명시적으로 표현
- Service를 통해 여러 Pod의 메트릭을 자동으로 수집

**실행**:

```bash
# values.yaml로 업그레이드
helm upgrade kube-prometheus-stack prometheus-community/kube-prometheus-stack -f values.yaml -n monitoring

# Prometheus Pod 확인
kubectl get pods -n monitoring | grep prometheus

# Prometheus에 접근
kubectl port-forward svc/kube-prometheus-stack-prometheus -n monitoring 9090:9090
```

**확인 방법**:

1. 브라우저에서 `http://localhost:9090` 접속
2. Status → Targets 메뉴에서 `hello-server` job 확인
3. Graph에서 `go_gc_duration_seconds{job="hello-server"}` 쿼리 실행

### 6단계: Grafana 설치 및 대시보드 구성

**목적**: 수집된 메트릭을 시각화하여 모니터링합니다.

**왜 Grafana를 사용하는가?**

- Prometheus와 완벽한 통합
- 풍부한 대시보드 템플릿
- 알림 설정 및 관리
- 사용자 친화적인 UI

#### 6-1. Grafana 접근

**자격 증명 확인**:

```bash
# Secret에서 자격 증명 확인
kubectl get secret kube-prometheus-stack-grafana -n monitoring -o jsonpath='{.data.admin-user}' | base64 -d
echo ""
kubectl get secret kube-prometheus-stack-grafana -n monitoring -o jsonpath='{.data.admin-password}' | base64 -d
echo ""

# 또는 환경 변수에서 확인
kubectl exec -n monitoring <grafana-pod-name> -- env | grep -i admin
```

**접근**:

```bash
# Grafana 포트 포워딩 (기본 포트: 3000)
kubectl port-forward svc/kube-prometheus-stack-grafana -n monitoring 3000:80

# 브라우저에서 http://localhost:3000 접속
```

#### 6-2. Prometheus 데이터소스 설정

**자동 설정 확인**:

- kube-prometheus-stack은 자동으로 Prometheus를 데이터소스로 등록합니다
- ConfigMap `kube-prometheus-stack-grafana-datasource`에서 확인 가능

**수동 설정 (필요 시)**:

1. Grafana UI → Configuration → Data Sources
2. Add data source → Prometheus 선택
3. URL: `http://kube-prometheus-stack-prometheus.monitoring.svc.cluster.local:9090`

**설계 이유**:

- Kubernetes DNS를 활용하여 서비스 이름으로 접근
- 네임스페이스(`monitoring`)를 명시하여 격리된 환경에서 동작

## 실습 파일 설명

### multinode-nodeport.yaml

**목적**: 멀티 노드 클러스터를 생성하고 포트 매핑을 설정합니다.

**구조**:

```yaml
nodes:
  - role: control-plane # 마스터 노드
  - role: worker # 워커 노드 1 (포트 매핑 있음)
    extraPortMappings:
      - containerPort: 30599 # 컨테이너 내부 포트
        hostPort: 30599 # 호스트 포트
        listenAddress: "127.0.0.1" # localhost만 리스닝
  - role: worker # 워커 노드 2 (포트 매핑 없음)
```

**설계 이유**:

- 첫 번째 worker에만 포트 매핑: 호스트 접근을 위한 최소 설정
- `listenAddress: "127.0.0.1"`: 보안을 위해 localhost만 접근 허용
- 두 번째 worker는 클러스터 내부에서만 접근 가능 (실제 환경과 유사)

### hello-server.yaml

**구성 요소**:

1. **Deployment**:

   - `namespace: develop`: 개발 환경 네임스페이스 사용
   - `replicas: 3`: 고가용성
   - `readinessProbe` / `livenessProbe`: `/healthz` 경로 사용
   - 리소스 제한 설정

2. **Service**:
   - `type: NodePort`: 외부 접근 가능
   - `port: 8080`: Service 포트
   - `targetPort: 8080`: Pod 포트

**설계 이유**:

- Health Check는 애플리케이션이 제공하는 `/healthz` 사용
- NodePort로 외부에서 접근 가능하도록 설정
- 리소스 제한으로 노드 보호

### values.yaml

**목적**: Prometheus의 커스텀 Scrape Config를 설정합니다.

**설정 내용**:

```yaml
prometheus:
  prometheusSpec:
    additionalScrapeConfigs:
      - job_name: hello-server
        scrape_interval: 10s
        static_configs:
          - targets:
              - hello-server.develop.svc.cluster.local:8080
```

**설계 이유**:

- `additionalScrapeConfigs`: 기본 설정에 추가로 커스텀 타겟 설정
- Kubernetes DNS 사용: Pod IP 변경에 영향받지 않음
- `scrape_interval: 10s`: 빠른 테스트를 위한 짧은 간격

### namespace.yaml

**목적**: 개발 환경 네임스페이스를 생성합니다.

**설계 이유**:

- 리소스 논리적 분리
- 권한 관리 및 격리
- 실제 프로덕션 환경 시뮬레이션

## 실습 단계별 상세 가이드

### 1. 클러스터 생성 및 확인

```bash
# 멀티 노드 클러스터 생성
kind create cluster --name multinode-nodeport --config multinode-nodeport.yaml --image=kindest/node:v1.29.0

# 노드 확인
kubectl get nodes -o wide

# 각 노드의 IP 확인
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.addresses[?(@.type=="InternalIP")].address}{"\n"}{end}'
```

**예상 출력**:

```
NAME                               INTERNAL-IP
multinode-nodeport-control-plane   172.18.0.3
multinode-nodeport-worker          172.18.0.4
multinode-nodeport-worker2         172.18.0.2
```

### 2. 네임스페이스 생성

```bash
# 네임스페이스 생성
kubectl apply -f namespace.yaml

# 확인
kubectl get namespace develop
```

### 3. 애플리케이션 배포

```bash
# Deployment 및 Service 생성
kubectl apply -f hello-server.yaml

# Pod 상태 확인 (다른 노드에 분산 배치됨)
kubectl get pods -n develop -o wide

# Service 확인
kubectl get svc -n develop

# NodePort 확인
kubectl get svc hello-server -n develop -o jsonpath='{.spec.ports[0].nodePort}'
```

**예상 결과**:

- 3개의 Pod가 서로 다른 노드에 배치됨
- NodePort가 자동으로 할당됨 (30000-32767 범위)

### 4. 애플리케이션 접근 테스트

```bash
# 방법 1: localhost로 접근 (포트 매핑이 있는 경우)
curl localhost:30599

# 방법 2: 클러스터 내부에서 모든 노드 IP로 접근 테스트
kubectl run test --image=curlimages/curl --rm -it -- sh -c "
  echo 'Testing 172.18.0.2:30599'
  curl 172.18.0.2:30599
  echo 'Testing 172.18.0.3:30599'
  curl 172.18.0.3:30599
  echo 'Testing 172.18.0.4:30599'
  curl 172.18.0.4:30599
"

# 방법 3: Service의 ClusterIP로 접근 (클러스터 내부)
kubectl run test --image=curlimages/curl --rm -it -- curl <cluster-ip>:8080
```

**학습 포인트**:

- NodePort Service는 모든 노드에서 접근 가능
- 클러스터 내부에서는 모든 노드 IP로 접근 가능
- 호스트 접근은 포트 매핑 설정에 따라 제한됨

### 5. Prometheus 설치

```bash
# Helm repo 추가
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# monitoring 네임스페이스 생성
kubectl create namespace monitoring

# Prometheus Stack 설치
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack -n monitoring

# 설치 상태 확인
kubectl get pods -n monitoring -w
```

**설치되는 컴포넌트**:

- Prometheus Operator
- Prometheus
- Grafana
- Node Exporter
- Kube State Metrics
- Alertmanager

### 6. 커스텀 Scrape Config 적용

```bash
# values.yaml로 업그레이드
helm upgrade kube-prometheus-stack prometheus-community/kube-prometheus-stack -f values.yaml -n monitoring

# Prometheus Pod 재시작 확인
kubectl get pods -n monitoring | grep prometheus

# Prometheus 접근
kubectl port-forward svc/kube-prometheus-stack-prometheus -n monitoring 9090:9090
```

**확인 방법**:

1. 브라우저에서 `http://localhost:9090` 접속
2. Status → Targets 메뉴
3. `hello-server` job이 UP 상태인지 확인
4. Graph에서 메트릭 쿼리:
   ```
   go_gc_duration_seconds{job="hello-server"}
   ```

### 7. Grafana 접근 및 설정

```bash
# Grafana 자격 증명 확인
kubectl get secret kube-prometheus-stack-grafana -n monitoring -o jsonpath='{.data.admin-password}' | base64 -d
echo ""

# Grafana 포트 포워딩
kubectl port-forward svc/kube-prometheus-stack-grafana -n monitoring 3000:80
```

**접근**:

1. 브라우저에서 `http://localhost:3000` 접속
2. Username: `admin`
3. Password: Secret에서 확인한 비밀번호

**대시보드 확인**:

- Home → Dashboards에서 기본 대시보드 확인
- Kubernetes / Compute Resources / Pods 등 다양한 대시보드 사용 가능

## 주요 개념 설명

### NodePort Service 동작 원리

**동작 흐름**:

```
1. 클라이언트가 <노드IP>:<NodePort>로 요청
2. 해당 노드의 kube-proxy가 요청 수신
3. kube-proxy가 Service의 Endpoints 확인
4. Endpoints 중 하나의 Pod로 트래픽 전달
5. Pod가 응답 반환
```

**특징**:

- 모든 노드에서 동일한 포트로 접근 가능
- 노드 간 트래픽 프록시는 kube-proxy가 담당
- Service의 `sessionAffinity` 설정에 따라 로드 밸런싱 방식 결정

### Kind 포트 매핑

**포트 매핑의 역할**:

- 호스트 포트 → 컨테이너 포트 연결
- 호스트에서 클러스터 내부 서비스 접근

**listenAddress의 의미**:

- `"127.0.0.1"`: localhost만 접근 가능 (보안)
- `"0.0.0.0"`: 모든 네트워크 인터페이스에서 접근 가능

**주의사항**:

- 포트 매핑은 호스트 접근을 위한 것
- 클러스터 내부에서는 포트 매핑 없이도 접근 가능

### Prometheus Scrape Config

**Scrape 과정**:

```
1. Prometheus가 설정된 타겟 목록 확인
2. 각 타겟의 /metrics 엔드포인트에 HTTP GET 요청
3. 응답받은 메트릭을 시계열 데이터베이스에 저장
4. scrape_interval에 따라 주기적으로 반복
```

**Kubernetes DNS 사용 이유**:

- Pod IP는 재시작 시 변경될 수 있음
- Service 이름은 안정적이며 자동으로 Pod IP로 해석됨
- 네임스페이스 분리를 명시적으로 표현

### Grafana 데이터소스

**자동 설정**:

- kube-prometheus-stack은 ConfigMap을 통해 자동으로 Prometheus를 데이터소스로 등록
- `kube-prometheus-stack-grafana-datasource` ConfigMap 확인

**수동 설정 시 URL**:

```
http://<service-name>.<namespace>.svc.cluster.local:<port>
```

## 트러블슈팅

### 1. NodePort 접근 불가

**증상**:

```bash
curl 172.18.0.4:30599
# Connection refused
```

**원인**:

- 포트 매핑이 `127.0.0.1`에만 바인딩됨
- 호스트에서 Docker 네트워크 IP로 직접 접근 불가

**해결**:

```bash
# localhost로 접근
curl localhost:30599

# 또는 포트 매핑 설정 변경 (클러스터 재생성 필요)
listenAddress: "0.0.0.0"
```

### 2. Prometheus에서 타겟이 Down 상태

**확인 방법**:

```bash
# Pod 상태 확인
kubectl get pods -n develop

# Service 확인
kubectl get svc -n develop

# Endpoints 확인
kubectl get endpoints -n develop hello-server

# Prometheus 로그 확인
kubectl logs -n monitoring prometheus-kube-prometheus-stack-prometheus-0 | grep hello-server
```

**가능한 원인**:

- Pod가 실행 중이 아님
- Service의 selector가 Pod label과 일치하지 않음
- 네트워크 정책으로 인한 접근 차단
- DNS 해석 실패

**해결**:

```bash
# Pod 재시작
kubectl rollout restart deployment hello-server -n develop

# Service selector 확인
kubectl get svc hello-server -n develop -o yaml | grep selector

# DNS 테스트
kubectl run test --image=busybox --rm -it -- nslookup hello-server.develop.svc.cluster.local
```

### 3. Grafana 로그인 실패

**확인 방법**:

```bash
# Secret에서 비밀번호 확인
kubectl get secret kube-prometheus-stack-grafana -n monitoring -o jsonpath='{.data.admin-password}' | base64 -d
echo ""

# 환경 변수 확인
kubectl exec -n monitoring <grafana-pod> -- env | grep GF_SECURITY_ADMIN_PASSWORD
```

**해결**:

- Secret의 비밀번호를 정확히 복사하여 사용
- 브라우저 캐시/쿠키 삭제
- Grafana Pod 재시작

### 4. 메트릭이 수집되지 않음

**확인 방법**:

```bash
# 애플리케이션의 /metrics 엔드포인트 확인
kubectl port-forward svc/hello-server -n develop 8080:8080
curl http://localhost:8080/metrics

# Prometheus 타겟 상태 확인
# 브라우저에서 http://localhost:9090/targets 접속
```

**가능한 원인**:

- 애플리케이션이 `/metrics` 엔드포인트를 제공하지 않음
- Scrape config의 타겟 URL이 잘못됨
- 네트워크 정책으로 인한 접근 차단

## 실습 단계 요약

1. ✅ 멀티 노드 클러스터 생성 (multinode-nodeport.yaml)
2. ✅ 네임스페이스 생성 (develop)
3. ✅ 애플리케이션 배포 (Deployment + Service)
4. ✅ NodePort Service 접근 테스트
5. ✅ Prometheus 설치 및 커스텀 설정 적용
6. ✅ 메트릭 수집 확인
7. ✅ Grafana 접근 및 대시보드 확인

## 다음 단계

- [ServiceMonitor 사용하기](#) - ServiceMonitor를 통한 자동 타겟 발견
- [알림 규칙 설정](#) - Alertmanager를 통한 알림 구성
- [커스텀 대시보드 생성](#) - Grafana에서 커스텀 대시보드 만들기

## 참고 자료

- [Kubernetes Service 문서](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Prometheus 공식 문서](https://prometheus.io/docs/)
- [Grafana 공식 문서](https://grafana.com/docs/)
- [Kind 포트 매핑 문서](https://kind.sigs.k8s.io/docs/user/configuration/#extra-port-mappings)
- [kube-prometheus-stack Helm Chart](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
