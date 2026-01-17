# 실습 환경 설정

# namespace 생성
kubectl create namespace echo-sound

# echoserver-deployment Deployment 생성
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echoserver-deployment
  namespace: echo-sound
spec:
  replicas: 1
  selector:
    matchLabels:
      app: echo
  template:
    metadata:
      labels:
        app: echo
    spec:
      containers:
      - name: echo
        image: hashicorp/http-echo:latest
        args:
          - -text="Echo Service"
          - -listen=:8080
        ports:
        - containerPort: 8080
EOF

# IngressClass 생성 (Ingress Controller가 없을 경우를 대비)
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
spec:
  controller: k8s.io/ingress-nginx
EOF

# Deployment와 Service 확인
kubectl get deployment echoserver-deployment -n echo-sound
kubectl get service echo-service -n echo-sound
kubectl get pods -n echo-sound

# IngressClass 확인
kubectl get ingressclass
