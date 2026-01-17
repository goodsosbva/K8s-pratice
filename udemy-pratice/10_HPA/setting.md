# 실습 환경 설정

# namespace 생성
kubectl create namespace autoscale

# apache-deployment 생성 (HPA가 작동하려면 resources.requests 설정 필요)
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: apache-deployment
  namespace: autoscale
spec:
  replicas: 1
  selector:
    matchLabels:
      app: apache
  template:
    metadata:
      labels:
        app: apache
    spec:
      containers:
      - name: apache
        image: httpd:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 256Mi
EOF

# Deployment 확인
kubectl get deployment apache-deployment -n autoscale
kubectl get pods -n autoscale

# metrics-server 확인 (HPA가 작동하려면 필요)
kubectl get deployment metrics-server -n kube-system 2>/dev/null || echo "⚠️  metrics-server가 설치되어 있지 않을 수 있습니다"

# HPA가 작동하는지 확인 (생성 후)
echo ""
echo "=== HPA 확인 방법 ==="
echo "kubectl get hpa apache-server -n autoscale"
echo "kubectl describe hpa apache-server -n autoscale"
