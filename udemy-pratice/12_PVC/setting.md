# 실습 환경 설정

# namespace 생성
kubectl create namespace mariadb

# 기존 Persistent Volume 확인
echo "=== 기존 Persistent Volume 확인 ==="
kubectl get pv

# Persistent Volume 생성 (재사용을 위해 Retain 정책)
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mariadb-pv
spec:
  capacity:
    storage: 500Mi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /mnt/data/mariadb
EOF

# maria.deploy.yaml 파일 생성 (초기에는 PVC 참조 없음, Deployment는 아직 생성 안 함)
cat > maria.deploy.yaml <<'DEPLOYMENT_EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: maria-deployment
  namespace: mariadb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mariadb
  template:
    metadata:
      labels:
        app: mariadb
    spec:
      containers:
      - name: mariadb
        image: mariadb:latest
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "password"
        ports:
        - containerPort: 3306
DEPLOYMENT_EOF

echo "maria.deploy.yaml 파일이 생성되었습니다 (초기에는 PVC 참조 없음)"
echo ""
echo "작업 순서:"
echo "1. PVC를 먼저 생성 (kubectl create pvc MariaDB ...)"
echo "   → PVC 상태: Pending (정상)"
echo "2. maria.deploy.yaml 파일을 수정하여 MariaDB PVC를 사용하도록 설정"
echo "   - volumeMounts 추가"
echo "   - volumes에 persistentVolumeClaim 추가"
echo "3. Deployment 적용 (kubectl apply -f maria.deploy.yaml)"
echo "   → PVC 상태: Bound ✅"
echo "   → Deployment가 실행되고 안정화됨"

# 확인
echo ""
echo "=== 현재 상태 확인 ==="
kubectl get pv
echo ""
echo "Deployment는 아직 생성되지 않았습니다 (파일만 준비됨)"
