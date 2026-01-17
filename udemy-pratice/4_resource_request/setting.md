# 실습 환경 설정

# Practice Test - Taints and Tolerations 파트에 노드가 2개 있음 실습 시, 이용할 것
# wordpress Deployment 생성 (init containers와 main containers 포함, 3 replicas)
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
spec:
  replicas: 3
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      initContainers:
        - name: init-wp
          image: busybox:latest
          command: ['sh', '-c', 'echo "Init container"']
      containers:
        - name: wordpress
          image: wordpress:latest
EOF

# Deployment 확인
kubectl get deployment wordpress
kubectl get pods -l app=wordpress
