# Practice Kubernetes

Kubernetes 학습을 위한 실습 프로젝트입니다.

## 프로젝트 구조

```
pratice-k8s/
├── .bashrc               # 프로젝트 전용 PATH 설정 (docker, kubectl)
├── hello-server/          # Go HTTP 서버 Docker 컨테이너
│   ├── main.go           # 서버 소스 코드
│   ├── Dockerfile        # Docker 빌드 설정
│   └── README.md         # 실행 정리
├── make-cluster/          # Kind를 사용한 Kubernetes 클러스터 구축
│   └── README.md         # 실행 정리
├── make-the-pod/         # Pod 생성 및 관리 실습
│   ├── myapp.yaml       # Pod 매니페스트 파일
│   └── README.md        # 실행 정리
├── try-debugging/        # Pod 디버깅 실습
│   ├── myapp.yaml       # 정상 Pod 매니페스트
│   ├── pod-destruction.yaml  # 문제가 있는 Pod 매니페스트
│   └── README.md        # 실행 정리
├── break-and-fix/       # 리소스 만들고 망가뜨리기 실습
│   ├── deployment-hello-server.yaml  # 정상 Deployment
│   ├── deployment-hello-server-rollingupdate.yaml  # 문제가 있는 Deployment
│   ├── service-nodeport.yaml  # NodePort Service
│   └── README.md        # 실행 정리
├── kind-config-portmapping.yaml  # 포트 매핑 설정
├── kind-config-multinode.yaml  # 멀티 노드 설정
├── kind-config-multinode-portmapping.yaml  # 멀티 노드 + 포트 매핑 설정
└── README.md             # 이 파일
```

## Quick Start

### 환경 설정 (Git Bash 사용 시)

프로젝트 디렉토리에서 다음 명령어로 docker와 kubectl을 인식할 수 있게 설정합니다:

```bash
source .bashrc
```

이 설정은 프로젝트 전용이며, 프로젝트를 삭제하면 설정도 함께 사라집니다. 자세한 내용은 [make-cluster 실행 정리](./make-cluster/README.md)를 참조하세요.

### hello-server 실행

```bash
# Docker 이미지 빌드
docker build ./hello-server --tag hello-server:1.0

# 컨테이너 실행
docker run --rm --detach --publish 8080:8080 --name hello-server hello-server:1.0

# 테스트
curl localhost:8080
```

### Kind 클러스터 생성

```bash
# 환경 설정 적용 (프로젝트 루트에서)
source .bashrc

# Kind 설치 확인
kind version

# 클러스터 생성
kind create cluster --image=kindest/node:v1.29.0

# 클러스터 확인
kubectl cluster-info --context kind-kind
kubectl get nodes

# 클러스터 삭제
kind delete cluster
```

**참고:** 프로젝트 루트의 `.bashrc` 파일을 `source`하면 docker와 kubectl을 자동으로 인식합니다. 자세한 내용은 [make-cluster 실행 정리](./make-cluster/README.md)를 참조하세요.

### Pod 생성

```bash
# make-the-pod 디렉토리로 이동
cd make-the-pod

# Pod 생성
kubectl apply -f myapp.yaml

# Pod 상태 확인
kubectl get pod
```

### Pod 디버깅

```bash
# try-debugging 디렉토리로 이동
cd try-debugging

# 문제가 있는 Pod 생성 (이미지 버전 오류)
kubectl apply -f pod-destruction.yaml

# Pod 상태 확인 (ErrImagePull 발생)
kubectl get pod myapp

# 문제 원인 확인
kubectl describe pod myapp

# Pod 수정 (이미지 버전 1.1 → 1.0)
kubectl edit pod myapp

# 수정 후 상태 확인
kubectl get pod myapp
```

### Deployment와 Service 실습

```bash
# break-and-fix 디렉토리로 이동
cd break-and-fix

# Deployment 생성
kubectl apply -f deployment-hello-server.yaml

# Pod 자동 복구 확인 (Pod 삭제 후 자동 재생성)
kubectl delete pod <pod-name>
kubectl get pods -l app=hello-server

# Rolling Update 실습 (문제 발생)
kubectl apply -f deployment-hello-server-rollingupdate.yaml
kubectl get pods -w  # ErrImagePull 발생 확인

# 문제 해결 (Deployment 수정)
kubectl edit deploy hello-server

# NodePort Service 생성
kubectl apply -f service-nodeport.yaml

# 포트 매핑 설정으로 클러스터 재생성 (접근 가능하도록)
kind delete cluster --name kind
kind create cluster --config ../kind-config-portmapping.yaml --image=kindest/node:v1.29.0

# 리소스 재생성 및 접근 테스트
kubectl apply -f deployment-hello-server.yaml
kubectl apply -f service-nodeport.yaml
curl localhost:30599
```

## 실행 정리

각 프로젝트의 실행 내용은 해당 디렉토리의 README.md 파일을 참조하세요.

- [hello-server 실행 정리](./hello-server/README.md)
- [make-cluster 실행 정리](./make-cluster/README.md)
- [make-the-pod 실행 정리](./make-the-pod/README.md)
- [try-debugging 실행 정리](./try-debugging/README.md)
- [break-and-fix 실행 정리](./break-and-fix/README.md)
