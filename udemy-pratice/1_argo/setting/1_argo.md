# 0. 설치

curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

helm version

# 1. 기본 도구 확인

kubectl version --client
helm version

# 2. 클러스터 연결 확인

kubectl cluster-info
kubectl get nodes

# 3. 필요한 디렉토리 생성

mkdir -p /home/argo

# 4. 네임스페이스 확인 (필요시 생성)

kubectl get namespace argocd || kubectl create namespace argocd

# 참고: Argo CD Helm Repository URL

# https://argoproj.github.io/argo-helm
