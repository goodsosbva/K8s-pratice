echo "=== 정답확인 (QUESTION 17) ==="

# namespace 확인
echo ""
echo "=== nginx-static namespace 확인 ==="
if kubectl get namespace nginx-static >/dev/null 2>&1; then
  echo "✅ nginx-static namespace 존재"
else
  echo "❌ nginx-static namespace를 찾을 수 없습니다"
  exit 1
fi

# Pod 상태 확인 (중요: Connection refused 원인 확인)
echo ""
echo "=== Pod 상태 확인 ==="
POD_NAME=$(kubectl get pods -n nginx-static -l app=nginx-static -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)
if [ -n "$POD_NAME" ]; then
  POD_STATUS=$(kubectl get pod $POD_NAME -n nginx-static -o jsonpath='{.status.phase}' 2>/dev/null)
  echo "Pod 이름: $POD_NAME"
  echo "Pod 상태: $POD_STATUS"
  
  if [ "$POD_STATUS" != "Running" ]; then
    echo "⚠️  Pod가 Running 상태가 아닙니다."
    echo ""
    echo "Pod 상세 정보:"
    kubectl describe pod $POD_NAME -n nginx-static | tail -30
    echo ""
    echo "Pod 로그 (가능한 경우):"
    kubectl logs $POD_NAME -n nginx-static --tail=20 2>/dev/null || echo "로그를 가져올 수 없습니다 (Pod가 아직 시작되지 않음)"
  else
    echo "✅ Pod가 Running 상태입니다"
    echo ""
    echo "Pod 내부에서 nginx 프로세스 확인:"
    kubectl exec $POD_NAME -n nginx-static -- ps aux | grep nginx 2>/dev/null || echo "nginx 프로세스 확인 불가"
    echo ""
    echo "Pod 내부에서 443 포트 리스닝 확인:"
    kubectl exec $POD_NAME -n nginx-static -- netstat -tlnp 2>/dev/null | grep 443 || \
    kubectl exec $POD_NAME -n nginx-static -- ss -tlnp 2>/dev/null | grep 443 || \
    echo "포트 확인 불가"
  fi
else
  echo "❌ Pod를 찾을 수 없습니다"
  kubectl get pods -n nginx-static
fi

# Service Endpoints 확인
echo ""
echo "=== Service Endpoints 확인 ==="
ENDPOINTS=$(kubectl get endpoints nginx-static -n nginx-static -o jsonpath='{.subsets[0].addresses[*].ip}' 2>/dev/null)
if [ -n "$ENDPOINTS" ]; then
  echo "✅ Service에 Endpoints가 있습니다: $ENDPOINTS"
else
  echo "❌ Service에 Endpoints가 없습니다 (Connection refused 원인)"
  echo "   Pod가 Ready 상태인지, Service selector가 Pod label과 일치하는지 확인하세요"
  kubectl get endpoints nginx-static -n nginx-static
fi

# Task 1: ConfigMap TLS 설정 확인
echo ""
echo "=== Task 1: ConfigMap TLS 설정 확인 ==="
# kube-root-ca.crt를 제외하고 nginx 관련 ConfigMap 찾기
CONFIGMAP_NAME=$(kubectl get configmap -n nginx-static -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}' 2>/dev/null | grep -v "kube-root-ca.crt" | head -1)

# ConfigMap 이름이 없으면 모든 ConfigMap 확인ㅉㅉ
if [ -z "$CONFIGMAP_NAME" ]; then
  CONFIGMAP_NAME=$(kubectl get configmap -n nginx-static -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)
fi

if [ -z "$CONFIGMAP_NAME" ]; then
  echo "❌ ConfigMap을 찾을 수 없습니다"
else
  echo "✅ ConfigMap 발견: $CONFIGMAP_NAME"
  
  # ConfigMap 내용 확인
  CONFIGMAP_DATA=$(kubectl get configmap $CONFIGMAP_NAME -n nginx-static -o jsonpath='{.data}' 2>/dev/null)
  
  # ssl_protocols 설정 확인
  SSL_PROTOCOLS=$(kubectl get configmap $CONFIGMAP_NAME -n nginx-static -o yaml | grep -i "ssl_protocols" | head -1)
  
  if [ -n "$SSL_PROTOCOLS" ]; then
    echo "ssl_protocols 설정: $SSL_PROTOCOLS"
    
    # TLSv1.3만 지원하는지 확인
    if echo "$SSL_PROTOCOLS" | grep -q "TLSv1.3" && ! echo "$SSL_PROTOCOLS" | grep -q "TLSv1.2"; then
      echo "✅ ConfigMap이 TLSv1.3만 지원하도록 설정되어 있습니다"
    elif echo "$SSL_PROTOCOLS" | grep -q "TLSv1.2"; then
      echo "❌ ConfigMap이 여전히 TLSv1.2를 지원합니다"
      echo "   TLSv1.2를 제거하고 TLSv1.3만 남겨야 합니다"
    else
      echo "⚠️  ssl_protocols 설정을 확인할 수 없습니다"
    fi
  else
    # ConfigMap의 모든 데이터 확인
    echo "ConfigMap 데이터 확인:"
    kubectl get configmap $CONFIGMAP_NAME -n nginx-static -o yaml | grep -A 50 "data:" | head -30
  fi
fi

# Task 2: /etc/hosts 확인
echo ""
echo "=== Task 2: /etc/hosts 확인 ==="
if grep -q "ITKiddie.k8s.local" /etc/hosts 2>/dev/null; then
  echo "✅ /etc/hosts에 ITKiddie.k8s.local 항목이 있습니다"
  HOSTS_ENTRY=$(grep "ITKiddie.k8s.local" /etc/hosts)
  echo "   항목: $HOSTS_ENTRY"
  
  # Service IP 확인
  SERVICE_IP=$(kubectl get service nginx-static -n nginx-static -o jsonpath='{.spec.clusterIP}' 2>/dev/null)
  HOSTS_IP=$(grep "ITKiddie.k8s.local" /etc/hosts | awk '{print $1}')
  
  if [ "$HOSTS_IP" = "$SERVICE_IP" ]; then
    echo "✅ /etc/hosts의 IP가 Service IP와 일치합니다 ($SERVICE_IP)"
  else
    echo "⚠️  /etc/hosts의 IP($HOSTS_IP)가 Service IP($SERVICE_IP)와 일치하지 않습니다"
  fi
else
  echo "❌ /etc/hosts에 ITKiddie.k8s.local 항목이 없습니다"
  SERVICE_IP=$(kubectl get service nginx-static -n nginx-static -o jsonpath='{.spec.clusterIP}' 2>/dev/null)
  if [ -n "$SERVICE_IP" ]; then
    echo "   Service IP: $SERVICE_IP"
    echo "   다음 명령어로 추가하세요:"
    echo "   echo '$SERVICE_IP ITKiddie.k8s.local' | sudo tee -a /etc/hosts"
  fi
fi

# Task 3: curl 테스트
echo ""
echo "=== Task 3: curl 테스트 ==="

# Service IP 확인
SERVICE_IP=$(kubectl get service nginx-static -n nginx-static -o jsonpath='{.spec.clusterIP}' 2>/dev/null)
SERVICE_PORT=$(kubectl get service nginx-static -n nginx-static -o jsonpath='{.spec.ports[0].port}' 2>/dev/null)

if [ -z "$SERVICE_IP" ] || [ -z "$SERVICE_PORT" ]; then
  echo "⚠️  Service IP 또는 Port를 찾을 수 없습니다"
else
  echo "Service IP: $SERVICE_IP"
  echo "Service Port: ${SERVICE_PORT:-443}"
  
  # TLSv1.2 테스트 (실패해야 함)
  echo ""
  echo "=== TLSv1.2 테스트 (실패해야 함) ==="
  TLS12_RESULT=$(curl --tls-max 1.2 https://ITKiddie.k8s.local -k -s -o /dev/null -w "%{http_code}" --connect-timeout 5 2>&1)
  
  if echo "$TLS12_RESULT" | grep -qE "^(0|000|35|51|60)" || echo "$TLS12_RESULT" | grep -qi "error\|refused\|timeout"; then
    echo "✅ TLSv1.2 연결 실패 (예상대로 작동함)"
    echo "   결과: $TLS12_RESULT"
  elif echo "$TLS12_RESULT" | grep -qE "^2[0-9]{2}"; then
    echo "❌ TLSv1.2 연결 성공 (실패해야 함 - ConfigMap 설정 확인 필요)"
    echo "   HTTP Status: $TLS12_RESULT"
  else
    echo "⚠️  TLSv1.2 테스트 결과: $TLS12_RESULT"
  fi
  
  # TLSv1.3 테스트 (성공해야 함)
  echo ""
  echo "=== TLSv1.3 테스트 (성공해야 함) ==="
  TLS13_RESULT=$(curl --tlsv1.3 https://ITKiddie.k8s.local -k -s -o /dev/null -w "%{http_code}" --connect-timeout 5 2>&1)
  
  if echo "$TLS13_RESULT" | grep -qE "^2[0-9]{2}"; then
    echo "✅ TLSv1.3 연결 성공"
    echo "   HTTP Status: $TLS13_RESULT"
  elif echo "$TLS13_RESULT" | grep -qE "^(0|000|35|51|60)" || echo "$TLS13_RESULT" | grep -qi "error\|refused\|timeout"; then
    echo "❌ TLSv1.3 연결 실패 (성공해야 함)"
    echo "   결과: $TLS13_RESULT"
  else
    echo "⚠️  TLSv1.3 테스트 결과: $TLS13_RESULT"
  fi
fi

# ConfigMap 상세 확인
echo ""
echo "=== ConfigMap 상세 정보 ==="
if [ -n "$CONFIGMAP_NAME" ]; then
  echo "ConfigMap 이름: $CONFIGMAP_NAME"
  echo ""
  echo "ConfigMap 전체 내용:"
  kubectl get configmap $CONFIGMAP_NAME -n nginx-static -o yaml | grep -A 100 "data:" | head -50
fi

# Service 상세 확인
echo ""
echo "=== Service 상세 정보 ==="
kubectl get service nginx-static -n nginx-static -o yaml | grep -A 10 "spec:"

# /etc/hosts 전체 확인
echo ""
echo "=== /etc/hosts 전체 내용 ==="
cat /etc/hosts | grep -E "ITKiddie|nginx-static" || echo "관련 항목 없음"

# 최종 확인
echo ""
echo "=== 최종 확인 ==="

TASK1_OK=false
TASK2_OK=false
TASK3_OK=false

# Task 1: ConfigMap TLSv1.3만 지원
if [ -n "$SSL_PROTOCOLS" ] && echo "$SSL_PROTOCOLS" | grep -q "TLSv1.3" && ! echo "$SSL_PROTOCOLS" | grep -q "TLSv1.2"; then
  TASK1_OK=true
fi

# Task 2: /etc/hosts 설정
if grep -q "ITKiddie.k8s.local" /etc/hosts 2>/dev/null; then
  HOSTS_IP=$(grep "ITKiddie.k8s.local" /etc/hosts | awk '{print $1}')
  SERVICE_IP=$(kubectl get service nginx-static -n nginx-static -o jsonpath='{.spec.clusterIP}' 2>/dev/null)
  if [ "$HOSTS_IP" = "$SERVICE_IP" ]; then
    TASK2_OK=true
  fi
fi

# Task 3: curl 테스트
if echo "$TLS12_RESULT" | grep -qE "^(0|000|35|51|60)" && echo "$TLS13_RESULT" | grep -qE "^2[0-9]{2}"; then
  TASK3_OK=true
fi

echo "1. ConfigMap TLSv1.3만 지원: $(if [ "$TASK1_OK" = true ]; then echo "✅"; else echo "❌"; fi)"
echo "2. /etc/hosts에 Service IP 추가: $(if [ "$TASK2_OK" = true ]; then echo "✅"; else echo "❌"; fi)"
echo "3. curl 테스트 (TLSv1.2 실패, TLSv1.3 성공): $(if [ "$TASK3_OK" = true ]; then echo "✅"; else echo "❌"; fi)"
