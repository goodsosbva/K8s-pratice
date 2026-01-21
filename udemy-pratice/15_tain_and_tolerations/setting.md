# 실습 환경 설정

# node01 존재 확인
echo "=== node01 노드 확인 ==="
if kubectl get node node01 >/dev/null 2>&1; then
  echo "✅ node01 노드 존재"
  kubectl get node node01
else
  echo "⚠️  node01 노드를 찾을 수 없습니다"
  echo "   노드 목록:"
  kubectl get nodes
fi

# 현재 node01의 taint 확인
echo ""
echo "=== node01 현재 taint 확인 ==="
kubectl describe node node01 | grep -A 5 Taints || echo "현재 taint 없음"

# 현재 노드 상태 확인
echo ""
echo "=== 현재 노드 상태 ==="
kubectl get nodes
