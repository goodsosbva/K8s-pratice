# 실습 환경 설정

# GatewayClass 생성 (문제에서 이미 설치되어 있다고 하지만 실습용)
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: nginx-class
spec:
  controllerName: example.com/gateway-controller
EOF

# 기존 Ingress 리소스 생성 
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - gateway.web.k8s.local
      secretName: web-tls
  rules:
    - host: gateway.web.k8s.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-service
                port:
                  number: 80
EOF

# TLS Secret 생성 (kubernetes.io/tls 타입)
kubectl create secret tls web-tls --cert=/dev/null --key=/dev/null --dry-run=client -o yaml | kubectl apply -f - 2>/dev/null || \
kubectl create secret generic web-tls --from-literal=tls.crt=placeholder --from-literal=tls.key=placeholder --dry-run=client -o yaml | kubectl apply -f -

# Backend Service 생성
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
EOF
