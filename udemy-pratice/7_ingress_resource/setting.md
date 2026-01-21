# 실습 환경 설정

# namespace 생성
kubectl create namespace echo-sound

# echoserver-deployment Deployment 생성
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echoserver-deployment
  namespace: echo-sound
spec:
  replicas: 1
  selector:
    matchLabels:
      app: echo
  template:
    metadata:
      labels:
        app: echo
    spec:
      containers:
      - name: echo
        image: hashicorp/http-echo:latest
        args:
          - -text="Echo Service"
          - -listen=:8080
        ports:
        - containerPort: 8080
EOF

# Ingress Controller 설치 확인 및 설치
echo "=== Ingress Controller 확인 ==="
if kubectl get pods -n ingress-nginx 2>/dev/null | grep -q "ingress-nginx-controller"; then
  echo "✅ Ingress Controller 이미 설치됨"
else
  echo "⚠️  Ingress Controller가 설치되어 있지 않습니다. 설치 중..."
  
  # Ingress Controller 설치 (NGINX)
  kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/cloud/deploy.yaml 2>/dev/null || \
  kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml 2>/dev/null
  
  if [ $? -eq 0 ]; then
    echo "✅ Ingress Controller 설치 완료"
    echo "   Pod가 Running 상태가 될 때까지 대기 중..."
    sleep 10
    kubectl wait --namespace ingress-nginx \
      --for=condition=ready pod \
      --selector=app.kubernetes.io/component=controller \
      --timeout=120s 2>/dev/null || echo "⚠️  Pod 시작 대기 중..."
  else
    echo "❌ Ingress Controller 설치 실패"
    echo "   수동 설치: kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/cloud/deploy.yaml"
  fi
fi

# IngressClass 생성 (없을 경우)
echo ""
echo "=== IngressClass 확인 ==="
if kubectl get ingressclass nginx >/dev/null 2>&1; then
  echo "✅ IngressClass nginx 존재"
else
  echo "⚠️  IngressClass nginx를 찾을 수 없습니다. 생성 중..."
  kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
spec:
  controller: k8s.io/ingress-nginx
EOF
  echo "✅ IngressClass nginx 생성 완료"
fi

# 리소스 상태 확인
echo ""
echo "=== 리소스 상태 확인 ==="
echo "Deployment:"
kubectl get deployment echoserver-deployment -n echo-sound
echo ""
echo "Pods:"
kubectl get pods -n echo-sound
echo ""
echo "Ingress Controller:"
kubectl get pods -n ingress-nginx 2>/dev/null || echo "Ingress Controller Pod 확인 중..."
echo ""
echo "Ingress Controller Service:"
kubectl get svc -n ingress-nginx 2>/dev/null | grep ingress || echo "Ingress Controller Service 확인 중..."
echo ""
echo "IngressClass:"
kubectl get ingressclass
echo ""
echo "=== 문제 해결을 위해 생성해야 할 리소스 ==="
echo "1. echo-service Service (NodePort, port 8080) - 사용자가 직접 생성해야 함"
echo "2. echo Ingress 리소스 - 사용자가 직접 생성해야 함"
echo ""
echo "현재 Service 상태:"
kubectl get service -n echo-sound 2>/dev/null || echo "Service 없음 (정상 - 사용자가 생성해야 함)"
echo ""
echo "현재 Ingress 상태:"
kubectl get ingress -n echo-sound 2>/dev/null || echo "Ingress 없음 (정상 - 사용자가 생성해야 함)"