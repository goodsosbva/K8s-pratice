# Practice Kubernetes

Kubernetes 학습을 위한 실습 프로젝트입니다.

## 프로젝트 구조

```
pratice-k8s/
├── .bashrc               # 프로젝트 전용 PATH 설정 (docker, kubectl)
├── hello-server/          # Go HTTP 서버 Docker 컨테이너
│   ├── main.go           # 서버 소스 코드
│   ├── Dockerfile        # Docker 빌드 설정
│   └── README.md         # 상세 가이드
├── make-cluster/          # Kind를 사용한 Kubernetes 클러스터 구축
│   └── README.md         # 상세 가이드
├── make-the-pod/         # Pod 생성 및 관리 실습
│   ├── myapp.yaml       # Pod 매니페스트 파일
│   └── README.md        # 상세 가이드
└── README.md             # 이 파일
```

## Quick Start

### 환경 설정 (Git Bash 사용 시)

프로젝트 디렉토리에서 다음 명령어로 docker와 kubectl을 인식할 수 있게 설정합니다:

```bash
source .bashrc
```

이 설정은 프로젝트 전용이며, 프로젝트를 삭제하면 설정도 함께 사라집니다. 자세한 내용은 [make-cluster 가이드](./make-cluster/README.md)를 참조하세요.

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

**참고:** 프로젝트 루트의 `.bashrc` 파일을 `source`하면 docker와 kubectl을 자동으로 인식합니다. 자세한 내용은 [make-cluster 가이드](./make-cluster/README.md)를 참조하세요.

### Pod 생성

```bash
# make-the-pod 디렉토리로 이동
cd make-the-pod

# Pod 생성
kubectl apply -f myapp.yaml

# Pod 상태 확인
kubectl get pod
```

## 상세 가이드

각 프로젝트의 상세한 설명은 해당 디렉토리의 README.md 파일을 참조하세요.

- [hello-server 상세 가이드](./hello-server/README.md)
- [make-cluster 상세 가이드](./make-cluster/README.md)
- [make-the-pod 상세 가이드](./make-the-pod/README.md)
