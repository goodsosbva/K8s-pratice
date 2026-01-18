echo "=== 정답확인 (QUESTION 15) ==="

# node01 존재 확인
echo ""
echo "=== node01 노드 확인 ==="
if kubectl get node node01 >/dev/null 2>&1; then
  echo "✅ node01 노드 존재"
else
  echo "❌ node01 노드를 찾을 수 없습니다"
  exit 1
fi

# node01의 taint 확인
echo ""
echo "=== node01 Taint 확인 ==="
TAINT_INFO=$(kubectl describe node node01 | grep -A 5 "Taints:" | tail -1)

if [ -z "$TAINT_INFO" ] || echo "$TAINT_INFO" | grep -q "none"; then
  echo "❌ node01에 taint가 설정되지 않았습니다"
else
  echo "✅ node01에 taint가 설정되어 있습니다"
  echo "   Taint 정보: $TAINT_INFO"
  
  # Taint 상세 확인
  TAINT_KEY=$(kubectl get node node01 -o jsonpath='{.spec.taints[0].key}' 2>/dev/null)
  TAINT_VALUE=$(kubectl get node node01 -o jsonpath='{.spec.taints[0].value}' 2>/dev/null)
  TAINT_EFFECT=$(kubectl get node node01 -o jsonpath='{.spec.taints[0].effect}' 2>/dev/null)
  
  echo ""
  echo "Taint 상세:"
  echo "  Key: ${TAINT_KEY:-없음}"
  echo "  Value: ${TAINT_VALUE:-없음}"
  echo "  Effect: ${TAINT_EFFECT:-없음}"
  
  # Taint 값 확인
  if [ "$TAINT_KEY" = "IT" ] && [ "$TAINT_VALUE" = "Kiddie" ] && [ "$TAINT_EFFECT" = "NoSchedule" ]; then
    echo ""
    echo "✅ Taint가 올바르게 설정되어 있습니다 (Key=IT, Value=Kiddie, Effect=NoSchedule)"
  else
    echo ""
    echo "❌ Taint가 올바르게 설정되지 않았습니다"
    echo "   기대값: Key=IT, Value=Kiddie, Effect=NoSchedule"
    echo "   실제값: Key=${TAINT_KEY:-없음}, Value=${TAINT_VALUE:-없음}, Effect=${TAINT_EFFECT:-없음}"
  fi
fi

# 일반 Pod가 node01에 스케줄되지 않는지 확인
echo ""
echo "=== 일반 Pod 스케줄링 테스트 ==="
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-normal-pod
spec:
  containers:
  - name: test
    image: busybox:latest
    command: ['sh', '-c', 'sleep 3600']
EOF

sleep 5
NORMAL_POD_NODE=$(kubectl get pod test-normal-pod -o jsonpath='{.spec.nodeName}' 2>/dev/null)
NORMAL_POD_STATUS=$(kubectl get pod test-normal-pod -o jsonpath='{.status.phase}' 2>/dev/null)

if [ "$NORMAL_POD_NODE" = "node01" ]; then
  echo "❌ 일반 Pod가 node01에 스케줄되었습니다 (taint가 제대로 작동하지 않음)"
else
  echo "✅ 일반 Pod가 node01에 스케줄되지 않았습니다"
  if [ -n "$NORMAL_POD_NODE" ]; then
    echo "   Pod가 다른 노드에 스케줄됨: $NORMAL_POD_NODE"
  else
    echo "   Pod 상태: ${NORMAL_POD_STATUS:-Unknown}"
  fi
fi

# 테스트 Pod 정리
kubectl delete pod test-normal-pod --ignore-not-found=true

# Toleration이 있는 Pod 확인
echo ""
echo "=== Toleration이 있는 Pod 확인 ==="
PODS_WITH_TOLERATION=$(kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.tolerations[*].key}{"\t"}{.spec.tolerations[*].value}{"\t"}{.spec.tolerations[*].effect}{"\n"}{end}' 2>/dev/null | grep -E "IT.*Kiddie.*NoSchedule|Kiddie.*IT.*NoSchedule")

if [ -z "$PODS_WITH_TOLERATION" ]; then
  echo "⚠️  Toleration이 설정된 Pod를 찾을 수 없습니다"
  echo "   모든 Pod 확인:"
  kubectl get pods -o wide
else
  echo "✅ Toleration이 설정된 Pod 발견:"
  echo "$PODS_WITH_TOLERATION"
fi

# node01에 스케줄된 Pod 확인
echo ""
echo "=== node01에 스케줄된 Pod 확인 ==="
PODS_ON_NODE01=$(kubectl get pods -o wide --field-selector spec.nodeName=node01 --no-headers 2>/dev/null | wc -l)

if [ "$PODS_ON_NODE01" -gt 0 ]; then
  echo "✅ node01에 Pod가 스케줄되어 있습니다 (개수: $PODS_ON_NODE01)"
  echo ""
  echo "node01의 Pod 목록:"
  kubectl get pods -o wide --field-selector spec.nodeName=node01
  
  # 각 Pod의 toleration 확인
  echo ""
  echo "=== node01의 Pod Toleration 확인 ==="
  for pod in $(kubectl get pods --field-selector spec.nodeName=node01 -o jsonpath='{.items[*].metadata.name}' 2>/dev/null); do
    TOLERATION_KEY=$(kubectl get pod $pod -o jsonpath='{.spec.tolerations[?(@.key=="IT")].key}' 2>/dev/null)
    TOLERATION_VALUE=$(kubectl get pod $pod -o jsonpath='{.spec.tolerations[?(@.key=="IT")].value}' 2>/dev/null)
    TOLERATION_EFFECT=$(kubectl get pod $pod -o jsonpath='{.spec.tolerations[?(@.key=="IT")].effect}' 2>/dev/null)
    
    if [ "$TOLERATION_KEY" = "IT" ] && [ "$TOLERATION_VALUE" = "Kiddie" ] && [ "$TOLERATION_EFFECT" = "NoSchedule" ]; then
      echo "✅ Pod '$pod'에 올바른 toleration이 설정되어 있습니다"
    else
      echo "⚠️  Pod '$pod'의 toleration 확인 필요"
      kubectl get pod $pod -o jsonpath='{.spec.tolerations[*]}' 2>/dev/null
      echo ""
    fi
  done
else
  echo "⚠️  node01에 Pod가 스케줄되지 않았습니다"
  echo "   Toleration이 설정된 Pod를 생성해야 합니다"
fi

# 최종 확인
echo ""
echo "=== 최종 확인 ==="

# 1. Taint 확인
TAINT_KEY=$(kubectl get node node01 -o jsonpath='{.spec.taints[0].key}' 2>/dev/null)
TAINT_VALUE=$(kubectl get node node01 -o jsonpath='{.spec.taints[0].value}' 2>/dev/null)
TAINT_EFFECT=$(kubectl get node node01 -o jsonpath='{.spec.taints[0].effect}' 2>/dev/null)

TAINT_OK=false
if [ "$TAINT_KEY" = "IT" ] && [ "$TAINT_VALUE" = "Kiddie" ] && [ "$TAINT_EFFECT" = "NoSchedule" ]; then
  TAINT_OK=true
  echo "✅ Taint 설정: Key=IT, Value=Kiddie, Effect=NoSchedule"
else
  echo "❌ Taint 설정 불완전: Key=${TAINT_KEY:-없음}, Value=${TAINT_VALUE:-없음}, Effect=${TAINT_EFFECT:-없음}"
fi

# 2. Toleration이 있는 Pod가 node01에 스케줄되었는지 확인
POD_ON_NODE01_WITH_TOLERATION=false
for pod in $(kubectl get pods --field-selector spec.nodeName=node01 -o jsonpath='{.items[*].metadata.name}' 2>/dev/null); do
  TOL_KEY=$(kubectl get pod $pod -o jsonpath='{.spec.tolerations[?(@.key=="IT")].key}' 2>/dev/null)
  TOL_VALUE=$(kubectl get pod $pod -o jsonpath='{.spec.tolerations[?(@.key=="IT")].value}' 2>/dev/null)
  TOL_EFFECT=$(kubectl get pod $pod -o jsonpath='{.spec.tolerations[?(@.key=="IT")].effect}' 2>/dev/null)
  
  if [ "$TOL_KEY" = "IT" ] && [ "$TOL_VALUE" = "Kiddie" ] && [ "$TOL_EFFECT" = "NoSchedule" ]; then
    POD_ON_NODE01_WITH_TOLERATION=true
    echo "✅ Pod '$pod'가 node01에 올바른 toleration으로 스케줄됨"
    break
  fi
done

if [ "$POD_ON_NODE01_WITH_TOLERATION" = false ]; then
  echo "❌ node01에 올바른 toleration을 가진 Pod가 스케줄되지 않았습니다"
fi

# 요약
echo ""
echo "=== 요구사항 확인 요약 ==="
echo "1. node01에 taint 추가 (Key=IT, Value=Kiddie, Type=NoSchedule): $(if [ "$TAINT_OK" = true ]; then echo "✅"; else echo "❌"; fi)"
echo "2. Toleration이 있는 Pod를 node01에 스케줄: $(if [ "$POD_ON_NODE01_WITH_TOLERATION" = true ]; then echo "✅"; else echo "❌"; fi)"
