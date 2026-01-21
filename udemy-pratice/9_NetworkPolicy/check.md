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
  
  # NetworkPolicy 전체 구조 확인
  echo ""
  echo "=== NetworkPolicy YAML 구조 확인 ==="
  kubectl get netpol $NETPOL_NAME -n backend -o yaml | grep -A 30 "spec:"
  
  # namespaceSelector 확인 (여러 방법으로 시도)
  NAMESPACE_SELECTOR1=$(kubectl get netpol $NETPOL_NAME -n backend -o jsonpath='{.spec.ingress[0].from[0].namespaceSelector.matchLabels.app}' 2>/dev/null)
  NAMESPACE_SELECTOR2=$(kubectl get netpol $NETPOL_NAME -n backend -o jsonpath='{.spec.ingress[0].from[1].namespaceSelector.matchLabels.app}' 2>/dev/null)
  
  if [ "$NAMESPACE_SELECTOR1" = "frontend" ] || [ "$NAMESPACE_SELECTOR2" = "frontend" ]; then
    echo "✅ frontend namespace 허용됨 (namespaceSelector: ${NAMESPACE_SELECTOR1:-$NAMESPACE_SELECTOR2})"
  else
    echo "⚠️  namespaceSelector 확인 필요"
    echo "   from[0].namespaceSelector: ${NAMESPACE_SELECTOR1:-없음}"
    echo "   from[1].namespaceSelector: ${NAMESPACE_SELECTOR2:-없음}"
  fi
  
  # podSelector 확인 (보호받는 Pod)
  POD_SELECTOR=$(kubectl get netpol $NETPOL_NAME -n backend -o jsonpath='{.spec.podSelector.matchLabels.app}' 2>/dev/null)
  if [ "$POD_SELECTOR" = "backend" ]; then
    echo "✅ podSelector가 backend Pod를 보호함"
  else
    echo "⚠️  podSelector 확인 필요 (현재: ${POD_SELECTOR:-없음})"
  fi
  
  # from 규칙의 podSelector 확인 (허용되는 Pod)
  FROM_POD_SELECTOR=$(kubectl get netpol $NETPOL_NAME -n backend -o jsonpath='{.spec.ingress[0].from[0].podSelector.matchLabels.app}' 2>/dev/null)
  FROM_POD_SELECTOR2=$(kubectl get netpol $NETPOL_NAME -n backend -o jsonpath='{.spec.ingress[0].from[1].podSelector.matchLabels.app}' 2>/dev/null)
  
  if [ "$FROM_POD_SELECTOR" = "frontend" ] || [ "$FROM_POD_SELECTOR2" = "frontend" ]; then
    echo "✅ frontend Pod 허용됨 (podSelector: ${FROM_POD_SELECTOR:-$FROM_POD_SELECTOR2})"
  else
    echo "⚠️  from 규칙의 podSelector 확인 필요"
    echo "   from[0].podSelector: ${FROM_POD_SELECTOR:-없음}"
    echo "   from[1].podSelector: ${FROM_POD_SELECTOR2:-없음}"
  fi
  
  # namespaceSelector와 podSelector가 같은 항목에 있는지 확인 (AND 조건)
  FROM_0_NS=$(kubectl get netpol $NETPOL_NAME -n backend -o jsonpath='{.spec.ingress[0].from[0].namespaceSelector.matchLabels.app}' 2>/dev/null)
  FROM_0_POD=$(kubectl get netpol $NETPOL_NAME -n backend -o jsonpath='{.spec.ingress[0].from[0].podSelector.matchLabels.app}' 2>/dev/null)
  
  if [ "$FROM_0_NS" = "frontend" ] && [ "$FROM_0_POD" = "frontend" ]; then
    echo "✅ namespaceSelector와 podSelector가 같은 항목에 있음 (AND 조건 - 올바름)"
  elif [ -n "$FROM_0_NS" ] && [ -z "$FROM_0_POD" ]; then
    echo "⚠️  from[0]에 namespaceSelector만 있고 podSelector가 없음"
  elif [ -z "$FROM_0_NS" ] && [ -n "$FROM_0_POD" ]; then
    echo "⚠️  from[0]에 podSelector만 있고 namespaceSelector가 없음"
  else
    echo "⚠️  from 구조 확인 필요"
  fi
  
  # 포트 확인
  PORT=$(kubectl get netpol $NETPOL_NAME -n backend -o jsonpath='{.spec.ingress[0].ports[0].port}' 2>/dev/null)
  if [ "$PORT" = "8080" ]; then
    echo "✅ 포트 8080 허용됨"
  else
    echo "⚠️  포트 확인 필요 (현재: ${PORT:-없음})"
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

# CNI 확인 (NetworkPolicy 지원 여부)
echo ""
echo "=== CNI 확인 (NetworkPolicy 지원) ==="
CNI_PODS=$(kubectl get pods -n kube-system -o jsonpath='{.items[*].metadata.name}' 2>/dev/null | grep -iE "flannel|calico|weave|canal" | wc -w)
if [ "$CNI_PODS" -gt 0 ]; then
  echo "✅ CNI 설치됨 (NetworkPolicy 지원 가능)"
  kubectl get pods -n kube-system | grep -iE "flannel|calico|weave|canal" | head -3
else
  echo "⚠️  CNI를 찾을 수 없습니다 (NetworkPolicy가 작동하지 않을 수 있음)"
  echo "   NetworkPolicy를 지원하는 CNI (Flannel, Calico 등)가 설치되어 있는지 확인하세요"
fi

# frontend에서 backend로 접근 가능한지 확인
echo ""
echo "=== frontend에서 backend로의 접근 허용 확인 ==="
if [ -n "$BACKEND_SERVICE" ]; then
  # Backend Pod 상태 확인
  echo "Backend Pod 상태 확인:"
  kubectl get pods -n backend -l app=backend
  BACKEND_POD_READY=$(kubectl get pods -n backend -l app=backend -o jsonpath='{.items[0].status.conditions[?(@.type=="Ready")].status}' 2>/dev/null)
  if [ "$BACKEND_POD_READY" != "True" ]; then
    echo "⚠️  Backend Pod가 Ready 상태가 아닙니다"
    kubectl describe pods -n backend -l app=backend | tail -20
  fi
  
  # Backend Service Endpoints 확인
  echo ""
  echo "Backend Service Endpoints 확인:"
  kubectl get endpoints -n backend $BACKEND_SERVICE
  
  # Frontend namespace에서 busybox Pod로 Backend 접근 시도
  echo ""
  echo "Frontend namespace에서 busybox Pod로 Backend 접근 시도 중..."
  FRONTEND_RESULT=$(kubectl run test-frontend -n frontend --image=busybox:latest --rm -i --restart=Never --timeout=10s -- wget -q -O- --timeout=5 http://${BACKEND_HOST}:${BACKEND_PORT} 2>&1)
  EXIT_CODE=$?
  
  if [ $EXIT_CODE -eq 0 ] && echo "$FRONTEND_RESULT" | grep -qi "nginx\|html\|200"; then
    echo "✅ frontend에서 backend로의 접근이 허용됨"
    echo "   응답: $(echo "$FRONTEND_RESULT" | head -3)"
  else
    echo "❌ frontend에서 backend로의 접근이 차단되거나 실패"
    echo "   종료 코드: $EXIT_CODE"
    echo "   결과: $FRONTEND_RESULT"
    echo ""
    echo "=== 상세 디버깅 정보 ==="
    echo "1. Backend Service:"
    kubectl get svc -n backend $BACKEND_SERVICE
    echo ""
    echo "2. Backend Pod 상태:"
    kubectl get pods -n backend -l app=backend -o wide
    echo ""
    echo "3. Backend Pod IP:"
    BACKEND_POD_IP=$(kubectl get pods -n backend -l app=backend -o jsonpath='{.items[0].status.podIP}' 2>/dev/null)
    echo "   $BACKEND_POD_IP"
    echo ""
    echo "4. NetworkPolicy 상세:"
    kubectl describe netpol $NETPOL_NAME -n backend | grep -A 15 "Allowing ingress"
    echo ""
    echo "5. Frontend namespace label 확인:"
    kubectl get namespace frontend --show-labels
    echo ""
    echo "6. Frontend Pod label 확인:"
    kubectl get pods -n frontend -l app=frontend --show-labels
    echo ""
    echo "7. Pod IP로 직접 접근 테스트:"
    if [ -n "$BACKEND_POD_IP" ]; then
      DIRECT_RESULT=$(kubectl run test-direct -n frontend --image=busybox:latest --rm -i --restart=Never --timeout=10s -- wget -q -O- --timeout=5 http://${BACKEND_POD_IP}:8080 2>&1)
      if [ $? -eq 0 ]; then
        echo "   ✅ Pod IP로 직접 접근 성공 (NetworkPolicy 문제일 수 있음)"
      else
        echo "   ❌ Pod IP로 직접 접근도 실패 (다른 문제일 수 있음)"
        echo "   결과: $DIRECT_RESULT"
      fi
      kubectl delete pod test-direct -n frontend >/dev/null 2>&1
    fi
  fi
  
  # 정리
  kubectl delete pod test-frontend -n frontend >/dev/null 2>&1
else
  echo "⚠️  Backend Service를 찾을 수 없습니다"
fi

# 정리
kubectl delete pod test-pod -n test-ns >/dev/null 2>&1
echo ""
echo "테스트 완료"
