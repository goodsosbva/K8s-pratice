echo "=== 정답확인 (QUESTION 10) ==="

# HPA 존재 확인
echo ""
echo "=== HPA 존재 확인 ==="
kubectl get hpa apache-server -n autoscale >/dev/null 2>&1 && echo "✅ HPA 존재" || echo "❌ HPA 없음"

# HPA 이름 확인
HPA_NAME=$(kubectl get hpa -n autoscale -o jsonpath='{.items[?(@.metadata.name=="apache-server")].metadata.name}' 2>/dev/null)
if [ "$HPA_NAME" = "apache-server" ]; then
  echo "✅ HPA 이름 올바름"
else
  echo "❌ HPA 이름 불일치 (현재: ${HPA_NAME:-없음})"
fi

# Target Deployment 확인
echo ""
echo "=== Target Deployment 확인 ==="
TARGET_DEPLOYMENT=$(kubectl get hpa apache-server -n autoscale -o jsonpath='{.spec.scaleTargetRef.name}' 2>/dev/null)
if [ "$TARGET_DEPLOYMENT" = "apache-deployment" ]; then
  echo "✅ Target Deployment 올바름 (apache-deployment)"
else
  echo "❌ Target Deployment 불일치 (현재: ${TARGET_DEPLOYMENT:-없음})"
fi

# Min Replicas 확인
echo ""
echo "=== Min Replicas 확인 ==="
MIN_REPLICAS=$(kubectl get hpa apache-server -n autoscale -o jsonpath='{.spec.minReplicas}' 2>/dev/null)
if [ "$MIN_REPLICAS" = "1" ]; then
  echo "✅ Min Replicas 올바름 (1)"
else
  echo "❌ Min Replicas 불일치 (현재: ${MIN_REPLICAS:-없음}, 기대값: 1)"
fi

# Max Replicas 확인
echo ""
echo "=== Max Replicas 확인 ==="
MAX_REPLICAS=$(kubectl get hpa apache-server -n autoscale -o jsonpath='{.spec.maxReplicas}' 2>/dev/null)
if [ "$MAX_REPLICAS" = "4" ]; then
  echo "✅ Max Replicas 올바름 (4)"
else
  echo "❌ Max Replicas 불일치 (현재: ${MAX_REPLICAS:-없음}, 기대값: 4)"
fi

# CPU Utilization 확인
echo ""
echo "=== CPU Utilization 확인 ==="
CPU_TARGET=$(kubectl get hpa apache-server -n autoscale -o jsonpath='{.spec.metrics[0].resource.target.averageUtilization}' 2>/dev/null)
if [ "$CPU_TARGET" = "50" ]; then
  echo "✅ CPU Utilization 올바름 (50%)"
else
  echo "❌ CPU Utilization 불일치 (현재: ${CPU_TARGET:-없음}, 기대값: 50)"
fi

# CPU Metric Type 확인
echo ""
echo "=== CPU Metric Type 확인 ==="
CPU_TYPE=$(kubectl get hpa apache-server -n autoscale -o jsonpath='{.spec.metrics[0].resource.target.type}' 2>/dev/null)
if [ "$CPU_TYPE" = "Utilization" ]; then
  echo "✅ CPU Metric Type 올바름 (Utilization)"
else
  echo "❌ CPU Metric Type 불일치 (현재: ${CPU_TYPE:-없음}, 기대값: Utilization)"
fi

# Downscale Stabilization Window 확인
echo ""
echo "=== Downscale Stabilization Window 확인 ==="
STABILIZATION=$(kubectl get hpa apache-server -n autoscale -o jsonpath='{.spec.behavior.scaleDown.stabilizationWindowSeconds}' 2>/dev/null)
if [ "$STABILIZATION" = "30" ]; then
  echo "✅ Downscale Stabilization Window 올바름 (30초)"
else
  echo "❌ Downscale Stabilization Window 불일치 (현재: ${STABILIZATION:-없음}, 기대값: 30)"
fi

# HPA 상세 정보
echo ""
echo "=== HPA 상세 정보 ==="
kubectl get hpa apache-server -n autoscale
echo ""
kubectl describe hpa apache-server -n autoscale

# Deployment의 resources.requests 확인 (HPA 작동을 위해 필요)
echo ""
echo "=== Deployment Resources 확인 ==="
HAS_REQUESTS=$(kubectl get deployment apache-deployment -n autoscale -o jsonpath='{.spec.template.spec.containers[0].resources.requests.cpu}' 2>/dev/null)
if [ -n "$HAS_REQUESTS" ]; then
  echo "✅ Deployment에 resources.requests 설정됨 (HPA 작동 가능)"
  echo "   CPU Request: $(kubectl get deployment apache-deployment -n autoscale -o jsonpath='{.spec.template.spec.containers[0].resources.requests.cpu}')"
else
  echo "⚠️  Deployment에 resources.requests가 없습니다. HPA가 제대로 작동하지 않을 수 있습니다."
fi
