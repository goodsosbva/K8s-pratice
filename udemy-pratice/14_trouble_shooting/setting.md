# 실습 환경 설정

# 문제 상황 생성: kube-apiserver가 etcd 포트 2380으로 잘못 설정되도록 함
echo "=== 문제 상황 생성 ==="

# kube-apiserver 매니페스트 파일 확인
if [ -f /etc/kubernetes/manifests/kube-apiserver.yaml ]; then
  echo "✅ kube-apiserver 매니페스트 파일 존재"
  
  # 현재 etcd 서버 설정 확인
  echo ""
  echo "현재 etcd 서버 설정:"
  grep -E "^\s*--etcd-servers=" /etc/kubernetes/manifests/kube-apiserver.yaml || echo "etcd 서버 설정을 찾을 수 없습니다"
  
  # etcd 포트를 2380으로 잘못 설정 (문제 상황 생성)
  echo ""
  echo "⚠️  문제 상황 생성: etcd 포트를 2380 (peer port)으로 잘못 설정합니다"
  sudo sed -i 's/:2379/:2380/g' /etc/kubernetes/manifests/kube-apiserver.yaml
  
  echo ""
  echo "변경된 etcd 서버 설정:"
  grep -E "^\s*--etcd-servers=" /etc/kubernetes/manifests/kube-apiserver.yaml
  
  echo ""
  echo "✅ 문제 상황 생성 완료: kube-apiserver가 etcd 포트 2380을 가리키도록 설정됨"
  echo "   kubelet이 자동으로 kube-apiserver Pod를 재시작합니다 (약 1-2분 소요)"
  echo "   이후 kube-apiserver가 시작되지 않는 문제가 발생합니다"
else
  echo "❌ kube-apiserver 매니페스트 파일을 찾을 수 없습니다"
  echo "   경로: /etc/kubernetes/manifests/kube-apiserver.yaml"
fi

# kube-apiserver Pod 상태 확인
echo ""
echo "=== kube-apiserver Pod 상태 확인 ==="
kubectl get pods -n kube-system | grep kube-apiserver 2>/dev/null || echo 