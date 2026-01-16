# 실습 환경 설정


# 1. wordpress deployment 생성 (실습용 최소 설정)
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
        - name: wordpress
          image: wordpress:latest
EOF

# 2. Deployment 확인
kubectl get deployment wordpress


