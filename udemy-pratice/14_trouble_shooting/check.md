echo "=== 정답확인 (QUESTION 14) ==="

# kube-apiserver Pod 상태 확인
echo ""
echo "=== kube-apiserver Pod 상태 확인 ==="
APISERVER_POD=$(kubectl get pods -n kube-system -o jsonpath='{.items[?(@.metadata.name=~"kube-apiserver.*")].metadata.name}' 2>/dev/null | head -1)

if [ -n "$APISERVER_POD" ]; then
  APISERVER_STATUS=$(kubectl get pod $APISERVER_POD -n kube-system -o jsonpath='{.status.phase}' 2>/dev/null)
  if [ "$APISERVER_STATUS" = "Running" ]; then
    echo "✅ kube-apiserver Pod가 Running 상태입니다"
  else
    echo "⚠️  kube-apiserver Pod 상태: ${APISERVER_STATUS:-Unknown}"
  fi
else
  echo "ℹ️  kube-apiserver가 정적 Pod일 수 있습니다"
fi

# kube-apiserver 매니페스트 파일 확인
echo ""
echo "=== kube-apiserver 매니페스트 파일 확인 ==="
if [ -f /etc/kubernetes/manifests/kube-apiserver.yaml ]; then
  echo "✅ kube-apiserver 매니페스트 파일 존재"
  
  # etcd 서버 설정 확인 (YAML 형식: - --etcd-servers=)
  ETCD_SERVERS=$(grep -E "^\s*-\s+--etcd-servers=" /etc/kubernetes/manifests/kube-apiserver.yaml | head -1)
  if [ -n "$ETCD_SERVERS" ]; then
    echo "etcd-servers 설정: $ETCD_SERVERS"
    
    # 포트 확인 (2380이 포함되어 있으면 오류)
    if echo "$ETCD_SERVERS" | grep -q ":2380"; then
      echo "❌ etcd 포트가 2380 (peer port)으로 설정되어 있습니다"
      echo "   올바른 포트는 2379 (client port)입니다"
    elif echo "$ETCD_SERVERS" | grep -q ":2379"; then
      echo "✅ etcd 포트가 2379 (client port)로 올바르게 설정되어 있습니다"
    else
      echo "⚠️  etcd 포트를 확인할 수 없습니다"
    fi
  else
    # 대체 패턴 시도 (공백 없이)
    ETCD_SERVERS=$(grep -E "--etcd-servers=" /etc/kubernetes/manifests/kube-apiserver.yaml | head -1)
    if [ -n "$ETCD_SERVERS" ]; then
      echo "etcd-servers 설정: $ETCD_SERVERS"
      if echo "$ETCD_SERVERS" | grep -q ":2380"; then
        echo "❌ etcd 포트가 2380 (peer port)으로 설정되어 있습니다"
        echo "   올바른 포트는 2379 (client port)입니다"
      elif echo "$ETCD_SERVERS" | grep -q ":2379"; then
        echo "✅ etcd 포트가 2379 (client port)로 올바르게 설정되어 있습니다"
      fi
    else
      echo "⚠️  --etcd-servers 설정을 찾을 수 없습니다"
    fi
  fi
  
  # etcd-servers-overrides 확인 (있는 경우)
  ETCD_SERVERS_OVERRIDES=$(grep -E "^\s*-\s+--etcd-servers-overrides=" /etc/kubernetes/manifests/kube-apiserver.yaml | head -1)
  if [ -z "$ETCD_SERVERS_OVERRIDES" ]; then
    ETCD_SERVERS_OVERRIDES=$(grep -E "--etcd-servers-overrides=" /etc/kubernetes/manifests/kube-apiserver.yaml | head -1)
  fi
  if [ -n "$ETCD_SERVERS_OVERRIDES" ]; then
    echo "etcd-servers-overrides 설정: $ETCD_SERVERS_OVERRIDES"
    if echo "$ETCD_SERVERS_OVERRIDES" | grep -q ":2380"; then
      echo "❌ etcd-servers-overrides에서 포트가 2380으로 설정되어 있습니다"
    fi
  fi
else
  echo "❌ kube-apiserver 매니페스트 파일을 찾을 수 없습니다"
  echo "   경로: /etc/kubernetes/manifests/kube-apiserver.yaml"
fi

# kube-apiserver 접근 가능 여부 확인
echo ""
echo "=== kube-apiserver 접근 가능 여부 확인 ==="
if kubectl cluster-info >/dev/null 2>&1; then
  echo "✅ kube-apiserver에 접근 가능합니다"
  kubectl cluster-info | head -1
else
  echo "❌ kube-apiserver에 접근할 수 없습니다"
  echo "   kubectl cluster-info 실패"
fi

# componentstatuses 확인
echo ""
echo "=== Component Status 확인 ==="
if kubectl get componentstatuses >/dev/null 2>&1; then
  echo "✅ Component Status 확인 가능"
  kubectl get componentstatuses 2>/dev/null | grep -E "NAME|scheduler|controller-manager" || echo "일부 컴포넌트 상태 확인"
else
  echo "⚠️  Component Status를 확인할 수 없습니다"
fi

# kube-apiserver 로그 확인 (에러 확인)
echo ""
echo "=== kube-apiserver 에러 로그 확인 ==="
if [ -n "$APISERVER_POD" ]; then
  ERROR_LOG=$(kubectl logs $APISERVER_POD -n kube-system --tail=50 2>/dev/null | grep -i "error\|fail\|2380" | head -5)
  if [ -n "$ERROR_LOG" ]; then
    echo "⚠️  에러 로그 발견:"
    echo "$ERROR_LOG"
  else
    echo "✅ 최근 에러 로그 없음"
  fi
else
  echo "ℹ️  정적 Pod 로그는 journalctl로 확인하세요:"
  echo "   sudo journalctl -u kubelet -n 100 | grep kube-apiserver"
fi

# etcd 포트 요약
echo ""
echo "=== etcd 포트 요약 ==="
echo "✅ 올바른 설정: etcd client port 2379"
echo "❌ 잘못된 설정: etcd peer port 2380"
echo ""
echo "kube-apiserver는 etcd와 통신할 때 client port (2379)를 사용해야 합니다."
echo "peer port (2380)는 etcd 클러스터 멤버 간 통신에만 사용됩니다."

# 최종 확인
echo ""
echo "=== 최종 확인 ==="
if [ -f /etc/kubernetes/manifests/kube-apiserver.yaml ]; then
  # etcd-servers 설정 확인 (여러 패턴 시도)
  ETCD_CHECK=$(grep -E "^\s*-\s+--etcd-servers=" /etc/kubernetes/manifests/kube-apiserver.yaml || grep -E "--etcd-servers=" /etc/kubernetes/manifests/kube-apiserver.yaml | head -1)
  
  if echo "$ETCD_CHECK" | grep -q ":2379" && ! echo "$ETCD_CHECK" | grep -q ":2380"; then
    echo "✅ kube-apiserver 매니페스트에 etcd 포트가 올바르게 설정되어 있습니다 (2379)"
  elif echo "$ETCD_CHECK" | grep -q ":2380"; then
    echo "❌ kube-apiserver 매니페스트에 etcd 포트가 잘못 설정되어 있습니다 (2380)"
    echo "   포트를 2380에서 2379로 변경하세요"
  else
    echo "⚠️  etcd 포트 설정을 확인할 수 없습니다"
  fi
fi

if kubectl cluster-info >/dev/null 2>&1; then
  echo "✅ kube-apiserver가 정상적으로 작동하고 있습니다"
else
  echo "❌ kube-apiserver가 작동하지 않습니다"
  echo "   매니페스트 파일을 수정한 후 kubelet이 자동으로 재시작할 때까지 대기하세요 (약 1-2분)"
fi