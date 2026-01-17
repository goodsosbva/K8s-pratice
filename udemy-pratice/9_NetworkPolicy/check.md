echo "=== 정답확인 (QUESTION 09) ==="

# NetworkPolicy 존재 확인
echo ""
echo "=== NetworkPolicy 존재 확인 ==="
NETPOL_COUNT=$(kubectl get netpol -n backend --no-headers 2>/dev/null | wc -l)
if [ "$NETPOL_COUNT" -gt 0 ]; then
  echo "✅ NetworkPolicy 존재 (개수: $NETPOL_COUNT)"
  kubectl get netpol -n backend
else
  echo "❌ NetworkPolicy 없음"
fi

# NetworkPolicy가 backend namespace에 있는지 확인
echo ""
echo "=== NetworkPolicy 위치 확인 ==="
NETPOL_NAMESPACE=$(kubectl get netpol -A -o jsonpath='{.items[?(@.metadata.namespace=="backend")].metadata.namespace}' 2>/dev/null | head -1)
if [ "$NETPOL_NAMESPACE" = "backend" ]; then
  echo "✅ NetworkPolicy가 backend namespace에 있음"
else
  echo "❌ NetworkPolicy가 backend namespace에 없음"
fi

# namespaceSelector로 frontend만 허용하는지 확인
echo ""
echo "=== NetworkPolicy 규칙 확인 ==="
NETPOL_NAME=$(kubectl get netpol -n backend -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)
if [ -n "$NETPOL_NAME" ]; then
  echo "NetworkPolicy 이름: $NETPOL_NAME"
  
  # namespaceSelector 확인
  NAMESPACE_SELECTOR=$(kubectl get netpol $NETPOL_NAME -n backend -o jsonpath='{.spec.ingress[0].from[0].namespaceSelector.matchLabels.name}' 2>/dev/null)
  if [ "$NAMESPACE_SELECTOR" = "frontend" ]; then
    echo "✅ frontend namespace만 허용됨"
  else
    echo "⚠️  namespaceSelector 확인 필요 (현재: ${NAMESPACE_SELECTOR:-없음})"
  fi
  
  # podSelector 확인 (모든 Pod에 적용되는지)
  POD_SELECTOR=$(kubectl get netpol $NETPOL_NAME -n backend -o jsonpath='{.spec.podSelector}' 2>/dev/null)
  if echo "$POD_SELECTOR" | grep -q "{}" || [ -z "$POD_SELECTOR" ]; then
    echo "✅ podSelector가 모든 Pod에 적용됨 (least permissive)"
  else
    echo "⚠️  podSelector 확인 필요"
  fi
  
  # policyTypes 확인
  POLICY_TYPES=$(kubectl get netpol $NETPOL_NAME -n backend -o jsonpath='{.spec.policyTypes[*]}' 2>/dev/null)
  if echo "$POLICY_TYPES" | grep -q "Ingress"; then
    echo "✅ Ingress 정책 타입 설정됨"
  else
    echo "⚠️  policyTypes 확인 필요"
  fi
  
  # NetworkPolicy 상세 정보
  echo ""
  echo "=== NetworkPolicy 상세 정보 ==="
  kubectl describe netpol $NETPOL_NAME -n backend
else
  echo "❌ NetworkPolicy를 찾을 수 없음"
fi

# least permissive 확인 (다른 namespace는 차단되는지)
echo ""
echo "=== Least Permissive 확인 ==="
echo "NetworkPolicy는 frontend namespace에서만 backend로의 트래픽을 허용해야 합니다"
echo "다른 namespace에서의 트래픽은 차단되어야 합니다 (least permissive)"

# 테스트용 namespace 생성 및 차단 확인
echo ""
echo "=== 다른 namespace에서의 접근 차단 테스트 ==="

# test-ns namespace 생성 (없으면)
if ! kubectl get namespace test-ns >/dev/null 2>&1; then
  echo "테스트용 namespace 생성 중..."
  kubectl create namespace test-ns
  sleep 2
fi

# test Pod 생성
echo "테스트 Pod 생성 중..."
kubectl run test-pod -n test-ns --image=busybox:latest --rm -i --restart=Never -- sh -c "sleep 3600" >/dev/null 2>&1 &
sleep 3

# backend Service 주소 확인
BACKEND_SERVICE=$(kubectl get svc -n backend -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)
if [ -n "$BACKEND_SERVICE" ]; then
  BACKEND_HOST="${BACKEND_SERVICE}.backend.svc.cluster.local"
  BACKEND_PORT=$(kubectl get svc $BACKEND_SERVICE -n backend -o jsonpath='{.spec.ports[0].port}' 2>/dev/null || echo "8080")
  
  echo "Backend Service: $BACKEND_HOST:$BACKEND_PORT"
  
  # test Pod에서 backend로 접근 시도
  echo "test-ns의 Pod에서 backend로 접근 시도 중..."
  TIMEOUT_RESULT=$(kubectl run test-connect -n test-ns --image=busybox:latest --rm -i --restart=Never --timeout=5s -- wget -T 2 -O- http://${BACKEND_HOST}:${BACKEND_PORT} 2>&1 || echo "TIMEOUT")
  
  if echo "$TIMEOUT_RESULT" | grep -qi "timeout\|connection refused\|no route to host"; then
    echo "✅ test-ns에서 backend로의 접근이 차단됨 (least permissive 확인)"
  else
    echo "⚠️  test-ns에서 backend로의 접근이 허용되거나 연결 실패 (NetworkPolicy 확인 필요)"
  fi
  
  # 정리
  kubectl delete pod test-connect -n test-ns >/dev/null 2>&1
else
  echo "⚠️  Backend Service를 찾을 수 없습니다"
fi

# frontend에서 backend로 접근 가능한지 확인
echo ""
echo "=== frontend에서 backend로의 접근 허용 확인 ==="
FRONTEND_POD=$(kubectl get pods -n frontend -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)
if [ -n "$FRONTEND_POD" ] && [ -n "$BACKEND_SERVICE" ]; then
  echo "Frontend Pod에서 Backend로 접근 시도 중..."
  FRONTEND_RESULT=$(kubectl exec -n frontend $FRONTEND_POD -- wget -T 2 -O- http://${BACKEND_HOST}:${BACKEND_PORT} 2>&1 || echo "FAILED")
  
  if echo "$FRONTEND_RESULT" | grep -qi "200\|connected\|nginx"; then
    echo "✅ frontend에서 backend로의 접근이 허용됨"
  else
    echo "⚠️  frontend에서 backend로의 접근이 차단되거나 실패 (NetworkPolicy 확인 필요)"
  fi
else
  echo "⚠️  Frontend Pod 또는 Backend Service를 찾을 수 없습니다"
fi

# 정리
kubectl delete pod test-pod -n test-ns >/dev/null 2>&1
echo ""
echo "테스트 완료"
