echo "=== 정답확인 (QUESTION 07) ==="

# Ingress 존재 확인
kubectl get ingress echo -n echo-sound >/dev/null 2>&1 && echo "✅ Ingress 존재" || echo "❌ Ingress 없음"

# Ingress 이름 확인
INGRESS_NAME=$(kubectl get ingress -n echo-sound -o jsonpath='{.items[?(@.metadata.name=="echo")].metadata.name}')
if [ "$INGRESS_NAME" = "echo" ]; then
  echo "✅ Ingress 이름 올바름"
else
  echo "❌ Ingress 이름 불일치 (현재: ${INGRESS_NAME:-없음})"
fi

# Service 존재 확인
kubectl get service echo-service -n echo-sound >/dev/null 2>&1 && echo "✅ Service 존재" || echo "❌ Service 없음"

# Service 타입 확인 (NodePort)
SERVICE_TYPE=$(kubectl get service echo-service -n echo-sound -o jsonpath='{.spec.type}')
if [ "$SERVICE_TYPE" = "NodePort" ]; then
  echo "✅ Service 타입 NodePort"
else
  echo "❌ Service 타입 불일치 (현재: ${SERVICE_TYPE:-없음})"
fi

# Service 포트 확인 (8080)
SERVICE_PORT=$(kubectl get service echo-service -n echo-sound -o jsonpath='{.spec.ports[0].port}')
if [ "$SERVICE_PORT" = "8080" ]; then
  echo "✅ Service 포트 8080"
else
  echo "❌ Service 포트 불일치 (현재: ${SERVICE_PORT:-없음})"
fi

# Ingress host 확인 (example.org)
INGRESS_HOST=$(kubectl get ingress echo -n echo-sound -o jsonpath='{.spec.rules[0].host}')
if [ "$INGRESS_HOST" = "example.org" ]; then
  echo "✅ Ingress host example.org"
else
  echo "❌ Ingress host 불일치 (현재: ${INGRESS_HOST:-없음})"
fi

# Ingress path 확인 (/echo 또는 /)
INGRESS_PATH=$(kubectl get ingress echo -n echo-sound -o jsonpath='{.spec.rules[0].http.paths[0].path}')
if [ "$INGRESS_PATH" = "/echo" ] || [ "$INGRESS_PATH" = "/" ]; then
  echo "✅ Ingress path 올바름 (현재: $INGRESS_PATH)"
else
  echo "❌ Ingress path 불일치 (현재: ${INGRESS_PATH:-없음})"
fi

# Ingress backend service 확인 (echo-service)
INGRESS_SERVICE=$(kubectl get ingress echo -n echo-sound -o jsonpath='{.spec.rules[0].http.paths[0].backend.service.name}')
if [ "$INGRESS_SERVICE" = "echo-service" ]; then
  echo "✅ Ingress backend service echo-service"
else
  echo "❌ Ingress backend service 불일치 (현재: ${INGRESS_SERVICE:-없음})"
fi

# Ingress backend service port 확인 (8080)
INGRESS_SERVICE_PORT=$(kubectl get ingress echo -n echo-sound -o jsonpath='{.spec.rules[0].http.paths[0].backend.service.port.number}')
if [ "$INGRESS_SERVICE_PORT" = "8080" ]; then
  echo "✅ Ingress backend service port 8080"
else
  echo "❌ Ingress backend service port 불일치 (현재: ${INGRESS_SERVICE_PORT:-없음})"
fi

# curl 테스트 각자 알아서
curl -o /dev/null -s -w "%{http_code}\n" -H "Host: example.org" http://192.168.117.34:32612/echo
200