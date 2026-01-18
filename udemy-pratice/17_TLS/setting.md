# 실습 환경 설정

# namespace 확인 및 생성
echo "=== nginx-static namespace 확인 ==="
if kubectl get namespace nginx-static >/dev/null 2>&1; then
  echo "✅ nginx-static namespace 존재"
else
  echo "⚠️  nginx-static namespace를 찾을 수 없습니다. 생성 중..."
  kubectl create namespace nginx-static
  echo "✅ nginx-static namespace 생성 완료"
fi

# TLS Secret 생성 (없을 경우)
echo ""
echo "=== TLS Secret 생성 ==="
if kubectl get secret nginx-tls -n nginx-static >/dev/null 2>&1; then
  echo "✅ TLS Secret 존재"
else
  echo "⚠️  TLS Secret을 찾을 수 없습니다. 생성 중..."
  # Self-signed 인증서 생성 (필수)
  if ! command -v openssl >/dev/null 2>&1; then
    echo "❌ openssl이 설치되어 있지 않습니다. TLS Secret 생성에 필요합니다."
    echo "   설치: apt-get update && apt-get install -y openssl"
  else
    # 임시 디렉토리에서 인증서 생성
    TEMP_DIR=$(mktemp -d)
    openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
      -keyout $TEMP_DIR/nginx.key \
      -out $TEMP_DIR/nginx.crt \
      -subj "/CN=ITKiddie.k8s.local" 2>/dev/null
    
    if [ -f $TEMP_DIR/nginx.crt ] && [ -f $TEMP_DIR/nginx.key ]; then
      kubectl create secret tls nginx-tls \
        --cert=$TEMP_DIR/nginx.crt \
        --key=$TEMP_DIR/nginx.key \
        -n nginx-static
      echo "✅ TLS Secret 생성 완료 (실제 인증서 사용)"
      rm -rf $TEMP_DIR
    else
      echo "❌ 인증서 생성 실패"
      rm -rf $TEMP_DIR
      exit 1
    fi
  fi
fi

# ConfigMap 생성 (TLSv1.2와 TLSv1.3 지원)
echo ""
echo "=== ConfigMap 생성 ==="
if kubectl get configmap nginx-config -n nginx-static >/dev/null 2>&1; then
  echo "✅ ConfigMap 존재"
  echo "현재 ConfigMap 내용:"
  kubectl get configmap nginx-config -n nginx-static -o yaml | grep -A 20 "ssl_protocols\|TLS" || echo "TLS 설정 확인 필요"
else
  echo "⚠️  ConfigMap을 찾을 수 없습니다. 생성 중..."
  kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: nginx-static
data:
  default.conf: |
    server {
        listen 443 ssl;
        server_name ITKiddie.k8s.local;
        
        ssl_certificate /etc/nginx/ssl/tls.crt;
        ssl_certificate_key /etc/nginx/ssl/tls.key;
        ssl_protocols TLSv1.2 TLSv1.3;
        
        location / {
            root /usr/share/nginx/html;
            index index.html;
        }
    }
EOF
  echo "✅ ConfigMap 생성 완료 (TLSv1.2, TLSv1.3 지원)"
fi

# nginx-static deployment 생성
echo ""
echo "=== nginx-static deployment 생성 ==="
if kubectl get deployment nginx-static -n nginx-static >/dev/null 2>&1; then
  echo "✅ nginx-static deployment 존재"
  kubectl get deployment nginx-static -n nginx-static
else
  echo "⚠️  nginx-static deployment를 찾을 수 없습니다. 생성 중..."
  kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-static
  namespace: nginx-static
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-static
  template:
    metadata:
      labels:
        app: nginx-static
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 443
          protocol: TCP
        volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/conf.d
          readOnly: true
        - name: tls-secret
          mountPath: /etc/nginx/ssl
          readOnly: true
      volumes:
      - name: nginx-config
        configMap:
          name: nginx-config
      - name: tls-secret
        secret:
          secretName: nginx-tls
          items:
          - key: tls.crt
            path: tls.crt
          - key: tls.key
            path: tls.key
EOF
  echo "✅ nginx-static deployment 생성 완료"
  echo "   Pod가 Running 상태가 될 때까지 대기 중..."
  sleep 10
  
  # Pod 상태 확인
  echo ""
  echo "Pod 상태 확인:"
  kubectl get pods -n nginx-static -l app=nginx-static
  
  # Pod가 시작되지 않으면 로그 확인
  POD_NAME=$(kubectl get pods -n nginx-static -l app=nginx-static -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)
  if [ -n "$POD_NAME" ]; then
    POD_STATUS=$(kubectl get pod $POD_NAME -n nginx-static -o jsonpath='{.status.phase}' 2>/dev/null)
    if [ "$POD_STATUS" != "Running" ]; then
      echo ""
      echo "⚠️  Pod가 아직 Running 상태가 아닙니다 (상태: $POD_STATUS)"
      echo "   Pod 이벤트 확인:"
      kubectl describe pod $POD_NAME -n nginx-static | grep -A 10 Events || echo "이벤트 없음"
    fi
  fi
fi

# Service 생성
echo ""
echo "=== nginx-static Service 생성 ==="
if kubectl get service nginx-static -n nginx-static >/dev/null 2>&1; then
  echo "✅ nginx-static service 존재"
  kubectl get service nginx-static -n nginx-static
else
  echo "⚠️  nginx-static service를 찾을 수 없습니다. 생성 중..."
  kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: nginx-static
  namespace: nginx-static
spec:
  selector:
    app: nginx-static
  ports:
  - port: 443
    targetPort: 443
    protocol: TCP
  type: ClusterIP
EOF
  echo "✅ nginx-static service 생성 완료"
fi

# Service IP 확인
echo ""
echo "=== Service IP 확인 ==="
SERVICE_IP=$(kubectl get service nginx-static -n nginx-static -o jsonpath='{.spec.clusterIP}' 2>/dev/null)
if [ -n "$SERVICE_IP" ]; then
  echo "Service IP: $SERVICE_IP"
else
  echo "⚠️  Service IP를 찾을 수 없습니다"
fi

# /etc/hosts 확인
echo ""
echo "=== /etc/hosts 확인 ==="
if grep -q "ITKiddie.k8s.local" /etc/hosts 2>/dev/null; then
  echo "✅ /etc/hosts에 ITKiddie.k8s.local 항목이 있습니다"
  grep "ITKiddie.k8s.local" /etc/hosts
else
  echo "⚠️  /etc/hosts에 ITKiddie.k8s.local 항목이 없습니다"
fi

# Pod 상태 확인
echo ""
echo "=== Pod 상태 확인 ==="
kubectl get pods -n nginx-static -l app=nginx-static 2>/dev/null || kubectl get pods -n nginx-static
