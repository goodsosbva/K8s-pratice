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

# Ingress Controller 확인 및 curl 테스트
echo ""
echo "=== Ingress Controller 확인 ==="

# 방법 1: IngressClass를 통해 찾기
INGRESS_CLASS=$(kubectl get ingressclass -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)
if [ -n "$INGRESS_CLASS" ]; then
  echo "✅ IngressClass 발견: $INGRESS_CLASS"
  INGRESS_CONTROLLER=$(kubectl get ingressclass $INGRESS_CLASS -o jsonpath='{.spec.controller}' 2>/dev/null)
  echo "   Controller: $INGRESS_CONTROLLER"
fi

# 방법 2: Ingress Controller Pod 찾기
INGRESS_POD_NS=$(kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\n"}{end}' | grep -iE "ingress|nginx" | head -1 | cut -f1)
INGRESS_POD_NAME=$(kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\n"}{end}' | grep -iE "ingress|nginx" | head -1 | cut -f2)

if [ -n "$INGRESS_POD_NS" ] && [ -n "$INGRESS_POD_NAME" ]; then
  echo "✅ Ingress Controller Pod 발견: $INGRESS_POD_NS/$INGRESS_POD_NAME"
fi

# 방법 3: Ingress Controller Service 찾기 (여러 네임스페이스에서)
INGRESS_NS=""
INGRESS_SVC=""

# 우선순위: ingress-nginx > kube-system > default > 기타
for ns in ingress-nginx kube-system default; do
  if kubectl get namespace $ns >/dev/null 2>&1; then
    SVC_LIST=$(kubectl get svc -n $ns -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}' 2>/dev/null | grep -iE "ingress|nginx")
    if [ -n "$SVC_LIST" ]; then
      INGRESS_SVC=$(echo "$SVC_LIST" | head -1)
      INGRESS_NS=$ns
      break
    fi
  fi
done

# 위 방법으로 못 찾으면 모든 네임스페이스에서 찾기
if [ -z "$INGRESS_NS" ] || [ -z "$INGRESS_SVC" ]; then
  ALL_SVCS=$(kubectl get svc -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\n"}{end}' 2>/dev/null)
  MATCHED=$(echo "$ALL_SVCS" | grep -iE "ingress|nginx" | head -1)
  if [ -n "$MATCHED" ]; then
    INGRESS_NS=$(echo "$MATCHED" | cut -f1)
    INGRESS_SVC=$(echo "$MATCHED" | cut -f2)
  fi
fi

# 디버깅 정보 출력
echo ""
echo "=== Ingress Controller 검색 결과 ==="
if [ -n "$INGRESS_NS" ] && [ -n "$INGRESS_SVC" ]; then
  echo "✅ Service 발견: $INGRESS_NS/$INGRESS_SVC"
else
  echo "❌❌❌ Ingress Controller가 설치되어 있지 않습니다! ❌❌❌"
  echo ""
  echo "=== 문제 진단 ==="
  echo "✅ IngressClass는 존재함: $INGRESS_CLASS"
  echo "❌ Ingress Controller Service 없음"
  echo "❌ Ingress Controller Pod 없음"
  echo ""
  echo "=== 해결 방법: Ingress Controller 설치 ==="
  echo "Ingress Controller를 설치해야 Ingress 리소스가 작동합니다."
  echo ""
  echo "방법 1: NGINX Ingress Controller 설치 (권장)"
  echo "  kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/cloud/deploy.yaml"
  echo ""
  echo "방법 2: 간단한 설치 (kind 환경용)"
  echo "  kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml"
  echo ""
  echo "설치 후 확인:"
  echo "  kubectl get pods -n ingress-nginx"
  echo "  kubectl get svc -n ingress-nginx"
  echo ""
  echo "=== 현재 상태 ==="
  echo "모든 Service:"
  kubectl get svc -A 2>/dev/null | head -10
  echo ""
  echo "Ingress Controller Pod:"
  kubectl get pods -A 2>/dev/null | grep -iE "ingress|nginx" || echo "  없음"
  echo ""
  echo "⚠️  Ingress Controller가 설치되지 않으면 Ingress 리소스가 작동하지 않습니다!"
  echo "⚠️  curl 테스트를 하려면 먼저 Ingress Controller를 설치하세요!"
fi

if [ -n "$INGRESS_NS" ] && [ -n "$INGRESS_SVC" ]; then
  echo "✅ Ingress Controller Service 발견: $INGRESS_NS/$INGRESS_SVC"
  INGRESS_NODEPORT=$(kubectl get svc $INGRESS_SVC -n $INGRESS_NS -o jsonpath='{.spec.ports[?(@.port==80)].nodePort}' 2>/dev/null)
  if [ -z "$INGRESS_NODEPORT" ]; then
    INGRESS_NODEPORT=$(kubectl get svc $INGRESS_SVC -n $INGRESS_NS -o jsonpath='{.spec.ports[0].nodePort}' 2>/dev/null)
  fi
  
  if [ -n "$INGRESS_NODEPORT" ]; then
    echo "✅ Ingress Controller NodePort: $INGRESS_NODEPORT"
    # 노드 IP 찾기
    NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}' 2>/dev/null)
    if [ -z "$NODE_IP" ]; then
      NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[0].address}' 2>/dev/null)
    fi
    
    if [ -n "$NODE_IP" ]; then
      echo "✅ 노드 IP: $NODE_IP"
      echo ""
      echo "=== curl 테스트 ==="
      echo "명령어: curl -o /dev/null -s -w \"%{http_code}\n\" -H \"Host: example.org\" http://$NODE_IP:$INGRESS_NODEPORT/echo"
      HTTP_CODE=$(curl -o /dev/null -s -w "%{http_code}\n" -H "Host: example.org" http://$NODE_IP:$INGRESS_NODEPORT/echo 2>/dev/null)
      if [ "$HTTP_CODE" = "200" ]; then
        echo "✅ curl 테스트 성공 (HTTP $HTTP_CODE)"
      else
        echo "❌ curl 테스트 실패 (HTTP $HTTP_CODE)"
        echo ""
        echo "=== 디버깅 정보 ==="
        echo "사용한 명령어: curl -H \"Host: example.org\" http://$NODE_IP:$INGRESS_NODEPORT/echo"
        echo ""
        echo "확인 사항:"
        echo "1. Ingress Controller가 실행 중인지 확인:"
        echo "   kubectl get pods -n $INGRESS_NS"
        echo ""
        echo "2. Ingress 리소스가 정상인지 확인:"
        echo "   kubectl describe ingress echo -n echo-sound"
        echo ""
        echo "3. Pod가 실행 중인지 확인:"
        echo "   kubectl get pods -n echo-sound"
        echo ""
        echo "4. Service Endpoints 확인:"
        echo "   kubectl get endpoints echo-service -n echo-sound"
        echo ""
        echo "=== 자주 하는 실수 ==="
        echo "❌ curl -H \"Host: example.org/echo\" http://172.20.249.158:30251"
        echo "   문제점:"
        echo "   1. Host 헤더에 경로 포함: Host는 도메인만 (example.org)"
        echo "   2. 잘못된 주소: 172.20.249.158:30251는 echo-service NodePort (Ingress Controller 아님)"
        echo ""
        echo "✅ 올바른 명령어:"
        echo "   curl -H \"Host: example.org\" http://$NODE_IP:$INGRESS_NODEPORT/echo"
        echo "   (경로 /echo는 URL에 포함, Host 헤더에는 도메인만)"
      fi
    else
      echo "⚠️  노드 IP를 찾을 수 없습니다"
    fi
  else
    echo "⚠️  Ingress Controller NodePort를 찾을 수 없습니다"
    echo "   Ingress Controller Service 확인: kubectl get svc -n $INGRESS_NS"
  fi
else
  echo "⚠️  Ingress Controller Service를 찾을 수 없습니다"
  echo "   Ingress Controller 설치 확인: kubectl get svc -A | grep ingress"
  echo ""
  echo "=== Ingress Controller 수동 찾기 ==="
  echo "1. 모든 Service 확인:"
  echo "   kubectl get svc -A | grep -E 'ingress|nginx'"
  echo ""
  echo "2. Ingress Controller Service의 NodePort 확인:"
  echo "   kubectl get svc <INGRESS_SVC_NAME> -n <INGRESS_NS> -o jsonpath='{.spec.ports[?(@.port==80)].nodePort}'"
  echo ""
  echo "3. 노드 IP 확인:"
  echo "   kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type==\"InternalIP\")].address}'"
  echo ""
  echo "=== 올바른 curl 명령어 ==="
  echo "형식: curl -H \"Host: example.org\" http://<NODE_IP>:<INGRESS_CONTROLLER_NODEPORT>/echo"
  echo ""
  echo "예시:"
  echo "  curl -o /dev/null -s -w \"%{http_code}\n\" -H \"Host: example.org\" http://192.168.117.34:32612/echo"
  echo ""
  echo "=== 잘못된 curl 명령어 예시 ==="
  echo "❌ curl -H \"example.org/echo\" https://172.20.249.158:30251"
  echo "   문제점:"
  echo "   1. Host 헤더 형식 오류: -H \"Host: example.org\" 형식이어야 함 (경로 포함 안됨)"
  echo "   2. 프로토콜 오류: https 대신 http 사용 (TLS 미설정 시)"
  echo "   3. 주소 오류: echo-service NodePort(30251)가 아닌 Ingress Controller 주소 사용"
  echo ""
  echo "❌ curl -H \"Host: example.org/echo\" https://172.20.249.158:30251"
  echo "   문제점:"
  echo "   1. Host 헤더에 경로 포함: Host 헤더는 도메인만 (example.org), 경로는 URL에 (/echo)"
  echo "   2. 프로토콜 오류: https 대신 http 사용"
  echo "   3. 주소 오류: echo-service NodePort가 아닌 Ingress Controller 주소 사용"
  echo ""
  echo "=== 핵심 포인트 ==="
  echo "1. Host 헤더: -H \"Host: example.org\" (도메인만, 경로 없음)"
  echo "2. URL 경로: /echo (URL에 포함)"
  echo "3. 프로토콜: http:// (TLS 미설정 시)"
  echo "4. 주소: Ingress Controller의 NodePort 사용 (echo-service NodePort 아님)"
  echo ""
  echo "=== Ingress Controller 찾는 방법 ==="
  echo "kubectl get svc -A | grep -E 'ingress|nginx'"
  echo "kubectl get svc -n ingress-nginx  # 일반적인 네임스페이스"
  echo "kubectl get svc -n kube-system | grep ingress"
fi