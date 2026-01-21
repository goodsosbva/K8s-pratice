# 실습 환경 설정

# namespace 생성
kubectl create namespace priority

# 기존 Priority Class 확인 (시스템 Priority Class 제외)
kubectl get priorityclass --field-selector=preemptionPolicy!=Never -o jsonpath='{.items[*].value}' | sort -n | tail -1

# 기존 user-defined Priority Class 생성 (예: 값 1000)
kubectl apply -f - <<EOF
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: existing-priority
value: 1000
description: "Existing priority class for testing"
EOF

# busybox-logger Deployment 생성
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: busybox-logger
  namespace: priority
spec:
  replicas: 1
  selector:
    matchLabels:
      app: busybox-logger
  template:
    metadata:
      labels:
        app: busybox-logger
    spec:
      containers:
      - name: busybox
        image: busybox:latest
        command: ['sh', '-c', 'while true; do echo "Logging..."; sleep 10; done']
EOF

# Deployment 확인
kubectl get deployment busybox-logger -n priority
kubectl get pods -n priority

# Priority Class 목록 확인
kubectl get priorityclass
