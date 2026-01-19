echo "=== 정답확인 (QUESTION 04) ==="

kubectl get deploy wordpress >/dev/null 2>&1 && echo "✅ Deployment 존재" || echo "❌ Deployment 없음"

kubectl get deploy wordpress -o jsonpath='{.spec.replicas}' | grep -qw '3' && echo "✅ Replicas 3 OK" || echo "❌ Replicas 불일치"

kubectl get deploy wordpress -o jsonpath='{.spec.template.spec.initContainers[*].name}' | grep -qw 'init-wp' && echo "✅ Init container 존재" || echo "❌ Init container 없음"

kubectl get deploy wordpress -o jsonpath='{.spec.template.spec.containers[*].name}' | grep -qw 'wordpress' && echo "✅ Main container 존재" || echo "❌ Main container 없음"

# Init container CPU requests 확인
INIT_CPU_REQ=$(kubectl get deploy wordpress -o jsonpath='{.spec.template.spec.initContainers[?(@.name=="init-wp")].resources.requests.cpu}' 2>/dev/null)
if [ -n "$INIT_CPU_REQ" ]; then
  echo "✅ Init container CPU requests 설정됨: $INIT_CPU_REQ"
else
  echo "❌ Init container CPU requests 없음"
fi

# Main container CPU requests 확인
MAIN_CPU_REQ=$(kubectl get deploy wordpress -o jsonpath='{.spec.template.spec.containers[?(@.name=="wordpress")].resources.requests.cpu}' 2>/dev/null)
if [ -n "$MAIN_CPU_REQ" ]; then
  echo "✅ Main container CPU requests 설정됨: $MAIN_CPU_REQ"
else
  echo "❌ Main container CPU requests 없음"
fi

# Init과 Main의 CPU requests가 같은지 확인 (값은 중요하지 않음, 같기만 하면 됨)
if [ -n "$INIT_CPU_REQ" ] && [ -n "$MAIN_CPU_REQ" ]; then
  if [ "$INIT_CPU_REQ" = "$MAIN_CPU_REQ" ]; then
    echo "✅ CPU requests 동일함 (Init: $INIT_CPU_REQ, Main: $MAIN_CPU_REQ)"
  else
    echo "❌ CPU requests 불일치 (Init: $INIT_CPU_REQ, Main: $MAIN_CPU_REQ)"
  fi
else
  echo "❌ CPU requests 비교 불가 (Init 또는 Main이 설정되지 않음)"
fi

# Init container Memory requests 확인
INIT_MEM_REQ=$(kubectl get deploy wordpress -o jsonpath='{.spec.template.spec.initContainers[?(@.name=="init-wp")].resources.requests.memory}' 2>/dev/null)
if [ -n "$INIT_MEM_REQ" ]; then
  echo "✅ Init container Memory requests 설정됨: $INIT_MEM_REQ"
else
  echo "❌ Init container Memory requests 없음"
fi

# Main container Memory requests 확인
MAIN_MEM_REQ=$(kubectl get deploy wordpress -o jsonpath='{.spec.template.spec.containers[?(@.name=="wordpress")].resources.requests.memory}' 2>/dev/null)
if [ -n "$MAIN_MEM_REQ" ]; then
  echo "✅ Main container Memory requests 설정됨: $MAIN_MEM_REQ"
else
  echo "❌ Main container Memory requests 없음"
fi

# Init과 Main의 Memory requests가 같은지 확인 (값은 중요하지 않음, 같기만 하면 됨)
if [ -n "$INIT_MEM_REQ" ] && [ -n "$MAIN_MEM_REQ" ]; then
  if [ "$INIT_MEM_REQ" = "$MAIN_MEM_REQ" ]; then
    echo "✅ Memory requests 동일함 (Init: $INIT_MEM_REQ, Main: $MAIN_MEM_REQ)"
  else
    echo "❌ Memory requests 불일치 (Init: $INIT_MEM_REQ, Main: $MAIN_MEM_REQ)"
  fi
else
  echo "❌ Memory requests 비교 불가 (Init 또는 Main이 설정되지 않음)"
fi

# Init container CPU limits 확인 (선택사항 - 설정되어 있으면 같아야 함)
INIT_CPU_LIM=$(kubectl get deploy wordpress -o jsonpath='{.spec.template.spec.initContainers[?(@.name=="init-wp")].resources.limits.cpu}' 2>/dev/null)
MAIN_CPU_LIM=$(kubectl get deploy wordpress -o jsonpath='{.spec.template.spec.containers[?(@.name=="wordpress")].resources.limits.cpu}' 2>/dev/null)

if [ -n "$INIT_CPU_LIM" ] && [ -n "$MAIN_CPU_LIM" ]; then
  echo "✅ Init container CPU limits 설정됨: $INIT_CPU_LIM"
  echo "✅ Main container CPU limits 설정됨: $MAIN_CPU_LIM"
  if [ "$INIT_CPU_LIM" = "$MAIN_CPU_LIM" ]; then
    echo "✅ CPU limits 동일함 (Init: $INIT_CPU_LIM, Main: $MAIN_CPU_LIM)"
  else
    echo "❌ CPU limits 불일치 (Init: $INIT_CPU_LIM, Main: $MAIN_CPU_LIM)"
  fi
elif [ -z "$INIT_CPU_LIM" ] && [ -z "$MAIN_CPU_LIM" ]; then
  echo "ℹ️  CPU limits 미설정 (udemy 환경에서는 정상)"
else
  echo "⚠️  CPU limits 일부만 설정됨 (Init: ${INIT_CPU_LIM:-없음}, Main: ${MAIN_CPU_LIM:-없음})"
fi

# Init container Memory limits 확인 (선택사항 - 설정되어 있으면 같아야 함)
INIT_MEM_LIM=$(kubectl get deploy wordpress -o jsonpath='{.spec.template.spec.initContainers[?(@.name=="init-wp")].resources.limits.memory}' 2>/dev/null)
MAIN_MEM_LIM=$(kubectl get deploy wordpress -o jsonpath='{.spec.template.spec.containers[?(@.name=="wordpress")].resources.limits.memory}' 2>/dev/null)

if [ -n "$INIT_MEM_LIM" ] && [ -n "$MAIN_MEM_LIM" ]; then
  echo "✅ Init container Memory limits 설정됨: $INIT_MEM_LIM"
  echo "✅ Main container Memory limits 설정됨: $MAIN_MEM_LIM"
  if [ "$INIT_MEM_LIM" = "$MAIN_MEM_LIM" ]; then
    echo "✅ Memory limits 동일함 (Init: $INIT_MEM_LIM, Main: $MAIN_MEM_LIM)"
  else
    echo "❌ Memory limits 불일치 (Init: $INIT_MEM_LIM, Main: $MAIN_MEM_LIM)"
  fi
elif [ -z "$INIT_MEM_LIM" ] && [ -z "$MAIN_MEM_LIM" ]; then
  echo "ℹ️  Memory limits 미설정 (udemy 환경에서는 정상)"
else
  echo "⚠️  Memory limits 일부만 설정됨 (Init: ${INIT_MEM_LIM:-없음}, Main: ${MAIN_MEM_LIM:-없음})"
fi