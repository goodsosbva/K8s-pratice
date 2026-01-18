echo "=== 정답확인 (QUESTION 16) ==="

# namespace 확인
CURRENT_NS=$(kubectl config view --minify -o jsonpath='{..namespace}' 2>/dev/null || echo "default")
echo "현재 namespace: ${CURRENT_NS:-default}"

# Deployment 존재 확인
echo ""
echo "=== nodeport-deployment 존재 확인 ==="
if kubectl get deployment nodeport-deployment -n ${CURRENT_NS:-default} >/dev/null 2>&1; then
  echo "✅ nodeport-deployment 존재"
else
  echo "❌ nodeport-deployment를 찾을 수 없습니다"
  exit 1
fi

# Task 1: Deployment 포트 설정 확인
echo ""
echo "=== Task 1: Deployment 포트 설정 확인 ==="
DEPLOYMENT_PORTS=$(kubectl get deployment nodeport-deployment -n ${CURRENT_NS:-default} -o jsonpath='{.spec.template.spec.containers[0].ports[*]}' 2>/dev/null)

if echo "$DEPLOYMENT_PORTS" | grep -q "80" && echo "$DEPLOYMENT_PORTS" | grep -q "TCP" && echo "$DEPLOYMENT_PORTS" | grep -q "http"; then
  echo "✅ Deployment에 포트 80, protocol TCP, name http가 설정되어 있습니다"
  
  # 상세 확인
  PORT_NUMBER=$(kubectl get deployment nodeport-deployment -n ${CURRENT_NS:-default} -o jsonpath='{.spec.template.spec.containers[0].ports[?(@.name=="http")].containerPort}' 2>/dev/null)
  PORT_PROTOCOL=$(kubectl get deployment nodeport-deployment -n ${CURRENT_NS:-default} -o jsonpath='{.spec.template.spec.containers[0].ports[?(@.name=="http")].protocol}' 2>/dev/null)
  PORT_NAME=$(kubectl get deployment nodeport-deployment -n ${CURRENT_NS:-default} -o jsonpath='{.spec.template.spec.containers[0].ports[?(@.name=="http")].name}' 2>/dev/null)
  
  if [ "$PORT_NUMBER" = "80" ] && [ "$PORT_PROTOCOL" = "TCP" ] && [ "$PORT_NAME" = "http" ]; then
    echo "   Port: $PORT_NUMBER, Protocol: $PORT_PROTOCOL, Name: $PORT_NAME"
  else
    echo "⚠️  포트 설정 상세 확인 필요 (Port: ${PORT_NUMBER:-없음}, Protocol: ${PORT_PROTOCOL:-없음}, Name: ${PORT_NAME:-없음})"
  fi
else
  echo "❌ Deployment에 포트 80, protocol TCP, name http가 올바르게 설정되지 않았습니다"
  echo "   현재 설정: $DEPLOYMENT_PORTS"
fi

# Service 존재 확인
echo ""
echo "=== Task 2 & 3: nodeport-service 존재 확인 ==="
if kubectl get service nodeport-service -n ${CURRENT_NS:-default} >/dev/null 2>&1; then
  echo "✅ nodeport-service 존재"
else
  echo "❌ nodeport-service를 찾을 수 없습니다"
  exit 1
fi

# Service 타입 확인 (NodePort)
echo ""
echo "=== Service 타입 확인 ==="
SERVICE_TYPE=$(kubectl get service nodeport-service -n ${CURRENT_NS:-default} -o jsonpath='{.spec.type}' 2>/dev/null)
if [ "$SERVICE_TYPE" = "NodePort" ]; then
  echo "✅ Service 타입이 NodePort입니다"
else
  echo "❌ Service 타입이 NodePort가 아닙니다 (현재: ${SERVICE_TYPE:-없음})"
fi

# Service 포트 설정 확인
echo ""
echo "=== Service 포트 설정 확인 ==="
SERVICE_PORT=$(kubectl get service nodeport-service -n ${CURRENT_NS:-default} -o jsonpath='{.spec.ports[0].port}' 2>/dev/null)
TARGET_PORT=$(kubectl get service nodeport-service -n ${CURRENT_NS:-default} -o jsonpath='{.spec.ports[0].targetPort}' 2>/dev/null)
NODE_PORT=$(kubectl get service nodeport-service -n ${CURRENT_NS:-default} -o jsonpath='{.spec.ports[0].nodePort}' 2>/dev/null)
PROTOCOL=$(kubectl get service nodeport-service -n ${CURRENT_NS:-default} -o jsonpath='{.spec.ports[0].protocol}' 2>/dev/null)

echo "Service Port: ${SERVICE_PORT:-없음}"
echo "Target Port: ${TARGET_PORT:-없음}"
echo "NodePort: ${NODE_PORT:-없음}"
echo "Protocol: ${PROTOCOL:-없음}"

# 포트 80 확인
if [ "$SERVICE_PORT" = "80" ] || [ "$TARGET_PORT" = "80" ]; then
  echo "✅ Service가 포트 80을 사용합니다"
else
  echo "⚠️  Service가 포트 80을 사용하지 않습니다"
fi

# TCP 프로토콜 확인
if [ "$PROTOCOL" = "TCP" ]; then
  echo "✅ Protocol이 TCP입니다"
else
  echo "⚠️  Protocol이 TCP가 아닙니다 (현재: ${PROTOCOL:-없음})"
fi

# NodePort 확인 (30000-32767 범위)
if [ -n "$NODE_PORT" ] && [ "$NODE_PORT" -ge 30000 ] && [ "$NODE_PORT" -le 32767 ]; then
  echo "✅ NodePort가 올바르게 설정되어 있습니다 ($NODE_PORT)"
else
  echo "⚠️  NodePort가 올바르게 설정되지 않았습니다 (현재: ${NODE_PORT:-없음})"
fi

# Service Selector 확인
echo ""
echo "=== Service Selector 확인 ==="
SERVICE_SELECTOR=$(kubectl get service nodeport-service -n ${CURRENT_NS:-default} -o jsonpath='{.spec.selector}' 2>/dev/null)
DEPLOYMENT_LABELS=$(kubectl get deployment nodeport-deployment -n ${CURRENT_NS:-default} -o jsonpath='{.spec.selector.matchLabels}' 2>/dev/null)

echo "Service Selector: $SERVICE_SELECTOR"
echo "Deployment Labels: $DEPLOYMENT_LABELS"

# Endpoints 확인
echo ""
echo "=== Service Endpoints 확인 ==="
ENDPOINTS=$(kubectl get endpoints nodeport-service -n ${CURRENT_NS:-default} -o jsonpath='{.subsets[0].addresses[*].ip}' 2>/dev/null)
ENDPOINTS_COUNT=$(echo "$ENDPOINTS" | wc -w)

if [ "$ENDPOINTS_COUNT" -gt 0 ]; then
  echo "✅ Service에 $ENDPOINTS_COUNT 개의 Endpoint가 연결되어 있습니다"
  echo "   Endpoints: $ENDPOINTS"
else
  echo "⚠️  Service에 Endpoint가 연결되지 않았습니다"
  echo "   Selector가 Deployment의 Pod label과 일치하는지 확인하세요"
fi

# 최종 확인
echo ""
echo "=== 최종 확인 ==="

TASK1_OK=false
TASK2_OK=false
TASK3_OK=false

# Task 1: Deployment 포트 설정
if [ "$PORT_NUMBER" = "80" ] && [ "$PORT_PROTOCOL" = "TCP" ] && [ "$PORT_NAME" = "http" ]; then
  TASK1_OK=true
fi

# Task 2: Service 생성 및 포트 80, TCP
if [ -n "$SERVICE_PORT" ] && [ "$PROTOCOL" = "TCP" ]; then
  TASK2_OK=true
fi

# Task 3: NodePort 타입
if [ "$SERVICE_TYPE" = "NodePort" ] && [ -n "$NODE_PORT" ]; then
  TASK3_OK=true
fi

echo "1. Deployment 포트 설정 (port 80, protocol TCP, name http): $(if [ "$TASK1_OK" = true ]; then echo "✅"; else echo "❌"; fi)"
echo "2. Service 생성 (container port 80, TCP): $(if [ "$TASK2_OK" = true ]; then echo "✅"; else echo "❌"; fi)"
echo "3. Service NodePort 타입 설정: $(if [ "$TASK3_OK" = true ]; then echo "✅"; else echo "❌"; fi)"

# Service 상세 정보
echo ""
echo "=== Service 상세 정보 ==="
kubectl get service nodeport-service -n ${CURRENT_NS:-default} -o yaml | grep -A 15 "spec:"
