echo "=== 정답확인 (QUESTION 04) ==="

kubectl get deploy wordpress >/dev/null 2>&1 && echo "✅ Deployment 존재" || echo "❌ Deployment 없음"

kubectl get deploy wordpress -o jsonpath='{.spec.replicas}' | grep -qw '3' && echo "✅ Replicas 3 OK" || echo "❌ Replicas 불일치"

kubectl get deploy wordpress -o jsonpath='{.spec.template.spec.initContainers[*].name}' | grep -qw 'init-wp' && echo "✅ Init container 존재" || echo "❌ Init container 없음"

kubectl get deploy wordpress -o jsonpath='{.spec.template.spec.containers[*].name}' | grep -qw 'wordpress' && echo "✅ Main container 존재" || echo "❌ Main container 없음"

# Init container CPU requests 확인
INIT_CPU_REQ=$(kubectl get deploy wordpress -o jsonpath='{.spec.template.spec.initContainers[?(@.name=="init-wp")].resources.requests.cpu}')
[ -n "$INIT_CPU_REQ" ] && echo "✅ Init container CPU requests 설정됨" || echo "❌ Init container CPU requests 없음"

# Main container CPU requests 확인
MAIN_CPU_REQ=$(kubectl get deploy wordpress -o jsonpath='{.spec.template.spec.containers[?(@.name=="wordpress")].resources.requests.cpu}')
[ -n "$MAIN_CPU_REQ" ] && echo "✅ Main container CPU requests 설정됨" || echo "❌ Main container CPU requests 없음"

# Init과 Main의 CPU requests가 같은지 확인
[ "$INIT_CPU_REQ" = "$MAIN_CPU_REQ" ] && [ -n "$INIT_CPU_REQ" ] && echo "✅ CPU requests 동일함" || echo "❌ CPU requests 불일치"

# Init container Memory requests 확인
INIT_MEM_REQ=$(kubectl get deploy wordpress -o jsonpath='{.spec.template.spec.initContainers[?(@.name=="init-wp")].resources.requests.memory}')
[ -n "$INIT_MEM_REQ" ] && echo "✅ Init container Memory requests 설정됨" || echo "❌ Init container Memory requests 없음"

# Main container Memory requests 확인
MAIN_MEM_REQ=$(kubectl get deploy wordpress -o jsonpath='{.spec.template.spec.containers[?(@.name=="wordpress")].resources.requests.memory}')
[ -n "$MAIN_MEM_REQ" ] && echo "✅ Main container Memory requests 설정됨" || echo "❌ Main container Memory requests 없음"

# Init과 Main의 Memory requests가 같은지 확인
[ "$INIT_MEM_REQ" = "$MAIN_MEM_REQ" ] && [ -n "$INIT_MEM_REQ" ] && echo "✅ Memory requests 동일함" || echo "❌ Memory requests 불일치"

# Init container CPU limits 확인
INIT_CPU_LIM=$(kubectl get deploy wordpress -o jsonpath='{.spec.template.spec.initContainers[?(@.name=="init-wp")].resources.limits.cpu}')
[ -n "$INIT_CPU_LIM" ] && echo "✅ Init container CPU limits 설정됨" || echo "❌ Init container CPU limits 없음"

# Main container CPU limits 확인
MAIN_CPU_LIM=$(kubectl get deploy wordpress -o jsonpath='{.spec.template.spec.containers[?(@.name=="wordpress")].resources.limits.cpu}')
[ -n "$MAIN_CPU_LIM" ] && echo "✅ Main container CPU limits 설정됨" || echo "❌ Main container CPU limits 없음"

# Init과 Main의 CPU limits가 같은지 확인
[ "$INIT_CPU_LIM" = "$MAIN_CPU_LIM" ] && [ -n "$INIT_CPU_LIM" ] && echo "✅ CPU limits 동일함" || echo "❌ CPU limits 불일치"

# Init container Memory limits 확인
INIT_MEM_LIM=$(kubectl get deploy wordpress -o jsonpath='{.spec.template.spec.initContainers[?(@.name=="init-wp")].resources.limits.memory}')
[ -n "$INIT_MEM_LIM" ] && echo "✅ Init container Memory limits 설정됨" || echo "❌ Init container Memory limits 없음"

# Main container Memory limits 확인
MAIN_MEM_LIM=$(kubectl get deploy wordpress -o jsonpath='{.spec.template.spec.containers[?(@.name=="wordpress")].resources.limits.memory}')
[ -n "$MAIN_MEM_LIM" ] && echo "✅ Main container Memory limits 설정됨" || echo "❌ Main container Memory limits 없음"

# Init과 Main의 Memory limits가 같은지 확인
[ "$INIT_MEM_LIM" = "$MAIN_MEM_LIM" ] && [ -n "$INIT_MEM_LIM" ] && echo "✅ Memory limits 동일함" || echo "❌ Memory limits 불일치"

## 실습 환경에 나온 계산된 메모리양 
메모리: 19243 Mi per pod
오버 헤드: 6414
리미트: 20000

CPU: 4770 MB per pod 
오버 헤드: 1590
리미트: 5000

3000
4000

14000
17000

udemy 환경에서는 리미트가 설정이 안되어야 정상 작동