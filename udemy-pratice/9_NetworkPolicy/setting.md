# 실습 환경 설정

# namespace 생성
kubectl create namespace frontend
kubectl create namespace backend

# namespace에 label 추가 (NetworkPolicy에서 namespaceSelector로 사용)
kubectl label namespace frontend app=frontend
kubectl label namespace backend app=backend

# Frontend Deployment 생성
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: nginx:latest
        ports:
        - containerPort: 80
EOF

# Backend Deployment 생성
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: nginx:latest
        ports:
        - containerPort: 8080
EOF

# Service 생성 (테스트용)
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: backend
spec:
  selector:
    app: backend
  ports:
  - port: 8080
    targetPort: 8080
EOF

# Deployment 확인
kubectl get deployment -n frontend
kubectl get deployment -n backend
kubectl get pods -n frontend
kubectl get pods -n backend

# namespace label 확인
kubectl get namespace frontend --show-labels
kubectl get namespace backend --show-labels
