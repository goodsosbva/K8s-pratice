echo "=== 정답확인 (QUESTION 11) ==="

# CNI 설치 확인
echo ""
echo "=== CNI 설치 확인 ==="

# Flannel 확인
FLANNEL_PODS=$(kubectl get pods -n kube-system -l app=flannel --no-headers 2>/dev/null | wc -l)
if [ "$FLANNEL_PODS" -gt 0 ]; then
  echo "✅ Flannel 설치됨 (Pod 개수: $FLANNEL_PODS)"
  kubectl get pods -n kube-system -l app=flannel
  CNI_TYPE="flannel"
elif kubectl get pods -n kube-system | grep -q "flannel"; then
  echo "✅ Flannel 설치됨"
  kubectl get pods -n kube-system | grep flannel
  CNI_TYPE="flannel"
else
  FLANNEL_PODS=0
fi

# Calico 확인 (calico-node Pod 또는 tigera-operator Pod 확인)
CALICO_NODE_PODS=$(kubectl get pods -n kube-system -l k8s-app=calico-node --no-headers 2>/dev/null | wc -l)
TIGERA_OPERATOR_PODS=$(kubectl get pods -n tigera-operator -l name=tigera-operator --no-headers 2>/dev/null | wc -l)

if [ "$CALICO_NODE_PODS" -gt 0 ]; then
  echo "✅ Calico 설치됨 (calico-node Pod 개수: $CALICO_NODE_PODS)"
  kubectl get pods -n kube-system -l k8s-app=calico-node
  CNI_TYPE="calico"
  CALICO_PODS=$CALICO_NODE_PODS
elif [ "$TIGERA_OPERATOR_PODS" -gt 0 ]; then
  echo "✅ Calico Operator 설치됨 (tigera-operator Pod 개수: $TIGERA_OPERATOR_PODS)"
  kubectl get pods -n tigera-operator -l name=tigera-operator
  echo "⚠️  Calico Custom Resources를 설치해야 실제 Calico Pod가 생성됩니다"
  echo "   kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.2/manifests/custom-resources.yaml"
  CNI_TYPE="calico"
  CALICO_PODS=0
elif kubectl get pods -n kube-system | grep -q "calico"; then
  echo "✅ Calico 설치됨"
  kubectl get pods -n kube-system | grep calico
  CNI_TYPE="calico"
  CALICO_PODS=1
else
  CALICO_PODS=0
fi

# CNI가 설치되지 않은 경우
if [ "$FLANNEL_PODS" -eq 0 ] && [ "$CALICO_PODS" -eq 0 ] && [ -z "$CNI_TYPE" ]; then
  echo "❌ CNI가 설치되어 있지 않습니다"
  echo "   Flannel 또는 Calico를 설치하세요"
  # exit 1 제거 (계속 확인 진행)
fi

# CNI DaemonSet 확인
echo ""
echo "=== CNI DaemonSet 확인 ==="
if [ "$CNI_TYPE" = "flannel" ]; then
  kubectl get daemonset -n kube-system | grep flannel
elif [ "$CNI_TYPE" = "calico" ]; then
  kubectl get daemonset -n kube-system | grep calico
fi

# Pod 간 통신 테스트
echo ""
echo "=== Pod 간 통신 테스트 ==="
POD1=$(kubectl get pods -n cni-test -l app=test-1 -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)
POD2=$(kubectl get pods -n cni-test -l app=test-2 -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)

if [ -n "$POD1" ] && [ -n "$POD2" ]; then
  POD2_IP=$(kubectl get pod $POD2 -n cni-test -o jsonpath='{.status.podIP}' 2>/dev/null)
  
  if [ -n "$POD2_IP" ]; then
    echo "Pod1: $POD1"
    echo "Pod2: $POD2 (IP: $POD2_IP)"
    echo "Pod1에서 Pod2로 ping 테스트 중..."
    
    PING_RESULT=$(kubectl exec -n cni-test $POD1 -- ping -c 2 $POD2_IP 2>&1)
    if echo "$PING_RESULT" | grep -q "2 packets transmitted, 2 received"; then
      echo "✅ Pod 간 통신 성공"
    else
      echo "⚠️  Pod 간 통신 실패 또는 확인 필요"
      echo "$PING_RESULT"
    fi
  else
    echo "⚠️  Pod2의 IP를 가져올 수 없습니다"
  fi
else
  echo "⚠️  테스트 Pod를 찾을 수 없습니다"
  echo "   세팅 스크립트를 실행하여 테스트 Pod를 생성하세요"
fi

# Network Policy 지원 확인
echo ""
echo "=== Network Policy 지원 확인 ==="
if [ "$CNI_TYPE" = "flannel" ]; then
  # Flannel은 기본적으로 Network Policy를 지원하지 않음
  # 하지만 문제에서는 Network Policy 지원을 요구하므로 확인 필요
  echo "⚠️  Flannel은 기본적으로 Network Policy를 지원하지 않습니다"
  echo "   Network Policy를 사용하려면 Calico를 선택하는 것이 좋습니다"
elif [ "$CNI_TYPE" = "calico" ]; then
  echo "✅ Calico는 Network Policy를 지원합니다"
  
  # NetworkPolicy 리소스가 사용 가능한지 확인
  if kubectl api-resources | grep -q networkpolicies; then
    echo "✅ NetworkPolicy 리소스 사용 가능"
  else
    echo "⚠️  NetworkPolicy 리소스를 찾을 수 없습니다"
  fi
fi

# CNI 버전 확인
echo ""
echo "=== CNI 버전 확인 ==="
if [ "$CNI_TYPE" = "flannel" ]; then
  FLANNEL_VERSION=$(kubectl get daemonset -n kube-system -l app=flannel -o jsonpath='{.items[0].spec.template.spec.containers[0].image}' 2>/dev/null | grep -oP 'v\d+\.\d+\.\d+' || echo "확인 불가")
  echo "Flannel 버전: $FLANNEL_VERSION"
  if echo "$FLANNEL_VERSION" | grep -q "v0.26.1"; then
    echo "✅ 올바른 버전 (v0.26.1)"
  else
    echo "⚠️  버전 확인 필요 (기대값: v0.26.1)"
  fi
elif [ "$CNI_TYPE" = "calico" ]; then
  CALICO_VERSION=$(kubectl get pods -n kube-system -l k8s-app=calico-node -o jsonpath='{.items[0].spec.containers[0].image}' 2>/dev/null | grep -oP 'v\d+\.\d+\.\d+' || echo "확인 불가")
  echo "Calico 버전: $CALICO_VERSION"
  if echo "$CALICO_VERSION" | grep -q "v3.28.2"; then
    echo "✅ 올바른 버전 (v3.28.2)"
  else
    echo "⚠️  버전 확인 필요 (기대값: v3.28.2)"
  fi
fi

# CNI Pod 상태 확인
echo ""
echo "=== CNI Pod 상태 확인 ==="
if [ "$CNI_TYPE" = "flannel" ]; then
  kubectl get pods -n kube-system -l app=flannel
  FAILED_PODS=$(kubectl get pods -n kube-system -l app=flannel --field-selector=status.phase!=Running --no-headers 2>/dev/null | wc -l || echo "0")
elif [ "$CNI_TYPE" = "calico" ]; then
  if [ "$CALICO_NODE_PODS" -gt 0 ]; then
    kubectl get pods -n kube-system -l k8s-app=calico-node
    FAILED_PODS=$(kubectl get pods -n kube-system -l k8s-app=calico-node --field-selector=status.phase!=Running --no-headers 2>/dev/null | wc -l || echo "0")
  else
    echo "Calico Node Pod가 아직 생성되지 않았습니다"
    kubectl get pods -n tigera-operator -l name=tigera-operator
    FAILED_PODS="0"
  fi
else
  FAILED_PODS="0"
fi

if [ -n "$FAILED_PODS" ] && [ "$FAILED_PODS" = "0" ]; then
  if [ "$CNI_TYPE" = "calico" ] && [ "$CALICO_NODE_PODS" -eq 0 ]; then
    echo "⚠️  Calico Operator는 설치되었지만 Custom Resources를 설치해야 합니다"
  else
    echo "✅ 모든 CNI Pod가 Running 상태"
  fi
else
  echo "⚠️  일부 CNI Pod가 Running 상태가 아닙니다 (개수: ${FAILED_PODS:-0})"
fi

# 요구사항 요약
echo ""
echo "=== 요구사항 확인 요약 ==="
if [ -n "$POD1" ] && [ -n "$POD2" ] && echo "$PING_RESULT" | grep -q "2 packets transmitted, 2 received" 2>/dev/null; then
  POD_COMM="✅"
else
  POD_COMM="⚠️"
fi
echo "1. Pod 간 통신: $POD_COMM"

if [ "$CNI_TYPE" = "calico" ]; then
  NETPOL_SUPPORT="✅ (Calico)"
elif [ "$CNI_TYPE" = "flannel" ]; then
  NETPOL_SUPPORT="⚠️ (Flannel은 기본적으로 미지원)"
else
  NETPOL_SUPPORT="❌"
fi
echo "2. Network Policy 지원: $NETPOL_SUPPORT"

if [ "$CNI_TYPE" = "flannel" ] || [ "$CNI_TYPE" = "calico" ]; then
  MANIFEST_INSTALL="✅"
else
  MANIFEST_INSTALL="❌"
fi
echo "3. Manifest로 설치: $MANIFEST_INSTALL"
