# Practice Kubernetes

Kubernetes 학습을 위한 실습 프로젝트입니다.

## 프로젝트 구조

```
pratice-k8s/
├── hello-server/          # Go HTTP 서버 Docker 컨테이너
│   ├── main.go           # 서버 소스 코드
│   ├── Dockerfile        # Docker 빌드 설정
│   └── README.md         # 상세 가이드
├── make-cluster/          # Kind를 사용한 Kubernetes 클러스터 구축
│   └── README.md         # 상세 가이드
└── README.md             # 이 파일
```

## Quick Start

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

```powershell
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

## 상세 가이드

각 프로젝트의 상세한 설명은 해당 디렉토리의 README.md 파일을 참조하세요.

- [hello-server 상세 가이드](./hello-server/README.md)
- [make-cluster 상세 가이드](./make-cluster/README.md)
