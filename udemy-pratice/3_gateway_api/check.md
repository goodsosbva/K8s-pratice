echo "=== 정답확인 (QUESTION 03) ==="

kubectl get gateway web-gateway >/dev/null 2>&1 && echo "✅ Gateway 존재" || echo "❌ Gateway 없음"

kubectl get gateway web-gateway -o jsonpath='{.spec.listeners[*].hostname}' | grep -qw 'gateway.web.k8s.local' && echo "✅ Gateway hostname OK" || echo "❌ Gateway hostname 불일치"

kubectl get gateway web-gateway -o jsonpath='{.spec.gatewayClassName}' | grep -qw 'nginx-class' && echo "✅ GatewayClass OK" || echo "❌ GatewayClass 불일치"

kubectl get gateway web-gateway -o jsonpath='{.spec.listeners[*].protocol}' | grep -qw 'HTTPS' && echo "✅ HTTPS listener OK" || echo "❌ HTTPS listener 없음"

kubectl get gateway web-gateway -o jsonpath='{.spec.listeners[*].port}' | grep -qw '443' && echo "✅ 포트 443 OK" || echo "❌ 포트 443 불일치"

kubectl get gateway web-gateway -o jsonpath='{.spec.listeners[*].tls.certificateRefs[*].name}' | grep -qw 'web-tls' && echo "✅ TLS 설정 OK" || echo "❌ TLS 설정 없음"

kubectl get httproute web-route >/dev/null 2>&1 && echo "✅ HTTPRoute 존재" || echo "❌ HTTPRoute 없음"

kubectl get httproute web-route -o jsonpath='{.spec.hostnames[*]}' | grep -qw 'gateway.web.k8s.local' && echo "✅ HTTPRoute hostname OK" || echo "❌ HTTPRoute hostname 불일치"

kubectl get httproute web-route -o jsonpath='{.spec.parentRefs[*].name}' | grep -qw 'web-gateway' && echo "✅ HTTPRoute Gateway 참조 OK" || echo "❌ HTTPRoute Gateway 참조 없음"

kubectl get httproute web-route -o jsonpath='{.spec.rules[*].matches[*].path.value}' | grep -qw '/' && echo "✅ HTTPRoute path 규칙 OK" || echo "❌ HTTPRoute path 규칙 없음"

kubectl get httproute web-route -o jsonpath='{.spec.rules[*].backendRefs[*].name}' | grep -qw 'web-service' && echo "✅ HTTPRoute backend service OK" || echo "❌ HTTPRoute backend service 불일치"

kubectl get httproute web-route -o jsonpath='{.spec.rules[*].backendRefs[*].port}' | grep -qw '80' && echo "✅ HTTPRoute backend port OK" || echo "❌ HTTPRoute backend port 불일치"
