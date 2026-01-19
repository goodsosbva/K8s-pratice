# 실습 환경 설정

# Gateway API CRD 설치 확인 및 설치
echo "=== Gateway API CRD 확인 ==="
if kubectl get crd gatewayclasses.gateway.networking.k8s.io >/dev/null 2>&1; then
  echo "✅ Gateway API CRD 설치됨"
else
  echo "⚠️  Gateway API CRD가 설치되어 있지 않습니다. 설치 중..."
  echo "   Gateway API v1.0.0 CRD 설치 중..."
  kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/standard-install.yaml 2>/dev/null
  if [ $? -eq 0 ]; then
    echo "✅ Gateway API CRD 설치 완료"
    # CRD가 적용될 때까지 잠시 대기
    echo "   CRD 적용 대기 중..."
    sleep 3
  else
    echo "❌ Gateway API CRD 설치 실패"
    echo "   수동 설치: kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/standard-install.yaml"
    exit 1
  fi
fi

# GatewayClass 확인 및 생성
echo ""
echo "=== GatewayClass 확인 ==="
if kubectl get gatewayclass nginx-class >/dev/null 2>&1; then
  echo "✅ GatewayClass nginx-class 존재"
else
  echo "⚠️  GatewayClass nginx-class를 찾을 수 없습니다. 생성 중..."
  kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: nginx-class
spec:
  controllerName: example.com/gateway-controller
EOF
  if [ $? -eq 0 ]; then
    echo "✅ GatewayClass nginx-class 생성 완료"
  else
    echo "❌ GatewayClass 생성 실패"
  fi
fi

# TLS Secret 생성 (실제 인증서 사용)
echo ""
echo "=== TLS Secret 생성 ==="
if kubectl get secret web-tls >/dev/null 2>&1; then
  echo "✅ TLS Secret web-tls 존재"
else
  echo "⚠️  TLS Secret을 찾을 수 없습니다. 생성 중..."
  # Self-signed 인증서 생성
  if command -v openssl >/dev/null 2>&1; then
    TEMP_DIR=$(mktemp -d)
    openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
      -keyout $TEMP_DIR/web.key \
      -out $TEMP_DIR/web.crt \
      -subj "/CN=gateway.web.k8s.local" 2>/dev/null
    
    if [ -f $TEMP_DIR/web.crt ] && [ -f $TEMP_DIR/web.key ]; then
      kubectl create secret tls web-tls \
        --cert=$TEMP_DIR/web.crt \
        --key=$TEMP_DIR/web.key
      echo "✅ TLS Secret 생성 완료 (실제 인증서 사용)"
      rm -rf $TEMP_DIR
    else
      echo "❌ 인증서 생성 실패"
      rm -rf $TEMP_DIR
    fi
  else
    echo "❌ openssl이 설치되어 있지 않습니다"
    echo "   설치: apt-get update && apt-get install -y openssl"
  fi
fi

# Backend Deployment 생성 (Service가 작동하려면 필요)
echo ""
echo "=== Backend Deployment 생성 ==="
if kubectl get deployment web-deployment >/dev/null 2>&1; then
  echo "✅ Backend Deployment 존재"
else
  echo "⚠️  Backend Deployment를 찾을 수 없습니다. 생성 중..."
  kubectl create deployment web-deployment --image=nginx:latest --replicas=1
  kubectl label deployment web-deployment app=web
  echo "✅ Backend Deployment 생성 완료"
fi

# Backend Service 생성
echo ""
echo "=== Backend Service 생성 ==="
if kubectl get service web-service >/dev/null 2>&1; then
  echo "✅ Backend Service 존재"
else
  echo "⚠️  Backend Service를 찾을 수 없습니다. 생성 중..."
  kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
EOF
  echo "✅ Backend Service 생성 완료"
fi

# 기존 Ingress 리소스 생성 (문제의 초기 상태)
echo ""
echo "=== 기존 Ingress 리소스 생성 ==="
if kubectl get ingress web >/dev/null 2>&1; then
  echo "✅ Ingress web 존재"
  echo "현재 Ingress 설정:"
  kubectl get ingress web -o yaml | grep -A 20 "spec:"
else
  echo "⚠️  Ingress web을 찾을 수 없습니다. 생성 중..."
  kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - gateway.web.k8s.local
      secretName: web-tls
  rules:
    - host: gateway.web.k8s.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-service
                port:
                  number: 80
EOF
  echo "✅ Ingress web 생성 완료"
fi

# 리소스 상태 확인
echo ""
echo "=== 리소스 상태 확인 ==="
echo "GatewayClass:"
kubectl get gatewayclass nginx-class 2>/dev/null || echo "  GatewayClass 없음"
echo ""
echo "Ingress:"
kubectl get ingress web
echo ""
echo "Service:"
kubectl get service web-service
echo ""
echo "Deployment:"
kubectl get deployment web-deployment
echo ""
echo "Pod:"
kubectl get pods -l app=web
