# 실습 환경 설정

# 현재 CNI 확인
echo "=== 현재 CNI 확인 ==="
kubectl get pods -n kube-system | grep -E "flannel|calico|weave|canal"

# Flannel 설치 (옵션 1)
echo ""
echo "=== Flannel 설치 ==="
kubectl apply -f https://github.com/flannel-io/flannel/releases/download/v0.26.1/kube-flannel.yml

# Calico 설치 (옵션 2) - Flannel 대신 사용하려면 주석 해제
# echo ""
# echo "=== Calico 설치 ==="
# kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.2/manifests/tigera-operator.yaml
# kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.2/manifests/custom-resources.yaml

# CNI 설치 확인
echo ""
echo "=== CNI 설치 확인 ==="
echo "CNI Pod가 Running 상태가 될 때까지 대기 중..."
sleep 10
kubectl get pods -n kube-system | grep -E "flannel|calico"
