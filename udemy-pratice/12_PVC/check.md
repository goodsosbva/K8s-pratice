echo "=== 정답확인 (QUESTION 12) ==="

# PVC 찾기 (MariaDB 또는 mariadb)
echo ""
echo "=== PVC 찾기 ==="
PVC_NAME=$(kubectl get pvc -n mariadb -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)
if [ -z "$PVC_NAME" ]; then
  echo "❌ PVC가 없습니다"
  exit 1
fi

echo "발견된 PVC: $PVC_NAME"

# PVC 존재 확인
echo ""
echo "=== PVC 존재 확인 ==="
if kubectl get pvc $PVC_NAME -n mariadb >/dev/null 2>&1; then
  echo "✅ PVC 존재 ($PVC_NAME)"
else
  echo "❌ PVC 없음"
  exit 1
fi

# PVC 이름 확인 (MariaDB인지 확인)
if [ "$PVC_NAME" = "MariaDB" ]; then
  echo "✅ PVC 이름 올바름 (MariaDB)"
else
  echo "⚠️  PVC 이름이 MariaDB가 아닙니다 (현재: $PVC_NAME)"
  echo "   문제에서는 MariaDB를 요구합니다"
fi

# Access Mode 확인
echo ""
echo "=== Access Mode 확인 ==="
ACCESS_MODE=$(kubectl get pvc $PVC_NAME -n mariadb -o jsonpath='{.spec.accessModes[0]}' 2>/dev/null)
if [ "$ACCESS_MODE" = "ReadWriteOnce" ]; then
  echo "✅ Access Mode 올바름 (ReadWriteOnce)"
else
  echo "❌ Access Mode 불일치 (현재: ${ACCESS_MODE:-없음}, 기대값: ReadWriteOnce)"
fi

# Storage Capacity 확인
echo ""
echo "=== Storage Capacity 확인 ==="
STORAGE=$(kubectl get pvc $PVC_NAME -n mariadb -o jsonpath='{.spec.resources.requests.storage}' 2>/dev/null)
if [ "$STORAGE" = "250Mi" ]; then
  echo "✅ Storage Capacity 올바름 (250Mi)"
else
  echo "❌ Storage Capacity 불일치 (현재: ${STORAGE:-없음}, 기대값: 250Mi)"
fi

# PVC 상태 확인
echo ""
echo "=== PVC 상태 확인 ==="
PVC_STATUS=$(kubectl get pvc $PVC_NAME -n mariadb -o jsonpath='{.status.phase}' 2>/dev/null)
if [ "$PVC_STATUS" = "Bound" ]; then
  echo "✅ PVC가 Bound 상태입니다"
  BOUND_PV=$(kubectl get pvc $PVC_NAME -n mariadb -o jsonpath='{.spec.volumeName}' 2>/dev/null)
  if [ -n "$BOUND_PV" ]; then
    echo "   Bound PV: $BOUND_PV"
  fi
elif [ "$PVC_STATUS" = "Pending" ]; then
  echo "⚠️  PVC가 Pending 상태입니다"
  echo "   Deployment에서 PVC를 사용하도록 설정하면 Bound 상태가 됩니다"
else
  echo "⚠️  PVC 상태: ${PVC_STATUS:-Unknown}"
fi

# Deployment가 PVC를 사용하는지 확인
echo ""
echo "=== Deployment PVC 사용 확인 ==="
DEPLOYMENT_PVC=$(kubectl get deployment maria-deployment -n mariadb -o jsonpath='{.spec.template.spec.volumes[0].persistentVolumeClaim.claimName}' 2>/dev/null)
if [ "$DEPLOYMENT_PVC" = "MariaDB" ]; then
  echo "✅ Deployment가 MariaDB PVC를 사용함"
elif [ "$DEPLOYMENT_PVC" = "$PVC_NAME" ]; then
  echo "⚠️  Deployment가 $PVC_NAME PVC를 사용 중 (문제에서는 MariaDB를 요구)"
else
  echo "❌ Deployment가 올바른 PVC를 사용하지 않음 (현재: ${DEPLOYMENT_PVC:-없음})"
fi

# Volume Mount 확인
echo ""
echo "=== Volume Mount 확인 ==="
MOUNT_PATH=$(kubectl get deployment maria-deployment -n mariadb -o jsonpath='{.spec.template.spec.containers[0].volumeMounts[0].mountPath}' 2>/dev/null)
if [ -n "$MOUNT_PATH" ]; then
  echo "✅ Volume Mount 설정됨 (경로: $MOUNT_PATH)"
else
  echo "⚠️  Volume Mount가 설정되지 않았습니다"
fi

# Deployment 상태 확인
echo ""
echo "=== Deployment 상태 확인 ==="
DEPLOYMENT_READY=$(kubectl get deployment maria-deployment -n mariadb -o jsonpath='{.status.readyReplicas}' 2>/dev/null)
DEPLOYMENT_DESIRED=$(kubectl get deployment maria-deployment -n mariadb -o jsonpath='{.spec.replicas}' 2>/dev/null)

if [ "$DEPLOYMENT_READY" = "$DEPLOYMENT_DESIRED" ] && [ "$DEPLOYMENT_READY" -gt 0 ]; then
  echo "✅ Deployment가 안정적으로 실행 중 (Ready: $DEPLOYMENT_READY/$DEPLOYMENT_DESIRED)"
else
  echo "⚠️  Deployment가 아직 안정적이지 않음 (Ready: ${DEPLOYMENT_READY:-0}/${DEPLOYMENT_DESIRED:-0})"
fi

# Pod 상태 확인
echo ""
echo "=== Pod 상태 확인 ==="
POD_STATUS=$(kubectl get pods -n mariadb -l app=mariadb -o jsonpath='{.items[0].status.phase}' 2>/dev/null)
if [ "$POD_STATUS" = "Running" ]; then
  echo "✅ Pod가 Running 상태입니다"
else
  echo "⚠️  Pod 상태: ${POD_STATUS:-Unknown}"
fi

# PVC 상세 정보
echo ""
echo "=== PVC 상세 정보 ==="
kubectl get pvc $PVC_NAME -n mariadb
kubectl describe pvc $PVC_NAME -n mariadb

# Deployment 상세 정보
echo ""
echo "=== Deployment 상세 정보 ==="
kubectl get deployment maria-deployment -n mariadb
kubectl describe deployment maria-deployment -n mariadb

# maria.deploy.yaml 파일 확인
echo ""
echo "=== maria.deploy.yaml 파일 확인 ==="
if [ -f maria.deploy.yaml ]; then
  echo "✅ maria.deploy.yaml 파일 존재"
  if grep -q "claimName: MariaDB" maria.deploy.yaml; then
    echo "✅ 파일에 MariaDB PVC 참조가 포함되어 있음"
    if grep -q "volumeMounts:" maria.deploy.yaml && grep -q "mountPath:" maria.deploy.yaml; then
      echo "✅ Volume Mount 설정도 포함되어 있음"
    else
      echo "⚠️  Volume Mount 설정이 없습니다"
    fi
  else
    echo "⚠️  파일에 MariaDB PVC 참조가 없습니다"
    echo "   maria.deploy.yaml 파일을 수정하여 MariaDB PVC를 사용하도록 설정하세요"
  fi
else
  echo "⚠️  maria.deploy.yaml 파일을 찾을 수 없습니다"
fi

# 작업 순서 확인
echo ""
echo "=== 작업 순서 확인 ==="
if [ "$PVC_STATUS" = "Pending" ] && [ "$DEPLOYMENT_PVC" != "MariaDB" ] && [ "$DEPLOYMENT_PVC" != "$PVC_NAME" ]; then
  echo "ℹ️  현재 상태:"
  echo "   - PVC가 Pending 상태 (정상)"
  echo "   - Deployment가 아직 PVC를 사용하지 않음"
  echo "   → maria.deploy.yaml을 수정하여 MariaDB PVC를 사용하도록 설정하세요"
elif [ "$PVC_STATUS" = "Bound" ] && [ "$DEPLOYMENT_PVC" = "MariaDB" ]; then
  echo "✅ 모든 작업이 완료되었습니다"
  echo "   - PVC가 Bound 상태"
  echo "   - Deployment가 MariaDB PVC를 사용 중"
elif [ "$PVC_STATUS" = "Bound" ] && [ "$DEPLOYMENT_PVC" = "$PVC_NAME" ] && [ "$PVC_NAME" != "MariaDB" ]; then
  echo "⚠️  PVC는 Bound 상태이지만 이름이 MariaDB가 아닙니다"
  echo "   - PVC 이름: $PVC_NAME"
  echo "   - 문제에서는 MariaDB를 요구합니다"
fi
