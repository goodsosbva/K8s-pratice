# 실습 환경 설정

# namespace 확인 (relative namespace = default 또는 현재 namespace)
echo "=== 현재 namespace 확인 ==="
CURRENT_NS=$(kubectl config view --minify -o jsonpath='{..namespace}' 2>/dev/null || echo "default")
echo "현재 namespace: ${CURRENT_NS:-default}"

# nodeport-deployment 존재 확인
echo ""
echo "=== nodeport-deployment 확인 ==="
if kubectl get deployment nodeport-deployment -n ${CURRENT_NS:-default} >/dev/null 2>&1; then
  echo "✅ nodeport-deployment 존재"
  kubectl get deployment nodeport-deployment -n ${CURRENT_NS:-default}
else
  echo "⚠️  nodeport-deployment를 찾을 수 없습니다"
  echo "   Deployment 생성 중..."
  
  # 기본 Deployment 생성 (이미지와 포트는 문제에 따라 수정 필요)
  kubectl create deployment nodeport-deployment --image=nginx:latest -n ${CURRENT_NS:-default} 2>/dev/null || \
  kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodeport-deployment
  namespace: ${CURRENT_NS:-default}
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nodeport
  template:
    metadata:
      labels:
        app: nodeport
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
          protocol: TCP
          name: http
EOF
  echo "✅ nodeport-deployment 생성 완료"
fi

# Deployment 상세 확인
echo ""
echo "=== nodeport-deployment 상세 정보 ==="
kubectl get deployment nodeport-deployment -n ${CURRENT_NS:-default} -o yaml | grep -A 10 "ports:" || echo "ports 설정 확인 필요"

# Pod 상태 확인
echo ""
echo "=== Pod 상태 확인 ==="
kubectl get pods -n ${CURRENT_NS:-default} -l app=nodeport 2>/dev/null || kubectl get pods -n ${CURRENT_NS:-default} | grep nodeport

# Service 확인
echo ""
echo "=== Service 확인 ==="
kubectl get service nodeport-service -n ${CURRENT_NS:-default} 2>/dev/null || echo "nodeport-service가 아직 생성되지 않았습니다"
