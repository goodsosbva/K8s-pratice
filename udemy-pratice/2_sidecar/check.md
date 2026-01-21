echo "=== 정답확인 (QUESTION 02) ==="

# 1. Deployment 존재 확인
kubectl get deploy wordpress >/dev/null 2>&1 && echo "✅ deployment 존재" || echo "❌ deployment 없음"

# 2. Sidecar 컨테이너 존재 확인 (containers에만 있어야 함)
echo ""
echo "=== Sidecar 컨테이너 확인 ==="
SIDECAR_EXISTS=$(kubectl get deploy wordpress -o jsonpath='{.spec.template.spec.containers[*].name}' 2>/dev/null | grep -qw sidecar && echo "yes" || echo "no")
if [ "$SIDECAR_EXISTS" = "yes" ]; then
  echo "✅ sidecar 존재 (containers에 있음)"
else
  echo "❌ sidecar 없음 (containers에 sidecar 컨테이너가 없습니다)"
  echo "   현재 containers: $(kubectl get deploy wordpress -o jsonpath='{.spec.template.spec.containers[*].name}' 2>/dev/null)"
fi

# 3. Sidecar 이미지 확인
echo ""
echo "=== Sidecar 이미지 확인 ==="
SIDECAR_IMAGE=$(kubectl get deploy wordpress -o jsonpath='{.spec.template.spec.containers[?(@.name=="sidecar")].image}' 2>/dev/null)
if [ "$SIDECAR_IMAGE" = "busybox:stable" ]; then
  echo "✅ 이미지 OK (busybox:stable)"
else
  echo "❌ 이미지 불일치 (현재: ${SIDECAR_IMAGE:-없음}, 기대값: busybox:stable)"
fi

# 4. Sidecar 명령어 확인 (sh -c 'tail -f /var/log/wordpress.log' 전체 확인)
echo ""
echo "=== Sidecar 명령어 확인 ==="
SIDECAR_COMMAND=$(kubectl get deploy wordpress -o jsonpath='{.spec.template.spec.containers[?(@.name=="sidecar")].command[*]}' 2>/dev/null)
FULL_COMMAND_STR=$(echo "$SIDECAR_COMMAND" | tr ' ' ' ')
# sh -c 'tail -f' 또는 sh -c 'tail -F' 패턴 확인
if echo "$FULL_COMMAND_STR" | grep -qE "sh.*-c.*tail -[fF].*wordpress.log"; then
  echo "✅ 커맨드 OK (sh -c 'tail -f /var/log/wordpress.log' 형식)"
  echo "   명령어: $SIDECAR_COMMAND"
else
  echo "❌ 커맨드 불일치"
  echo "   현재: $SIDECAR_COMMAND"
  echo "   기대: sh -c 'tail -f /var/log/wordpress.log' 또는 sh -c 'tail -F /var/log/wordpress.log'"
fi

# 5. 볼륨 확인
echo ""
echo "=== 볼륨 확인 ==="
VOLUME_EXISTS=$(kubectl get deploy wordpress -o jsonpath='{.spec.template.spec.volumes[*].name}' 2>/dev/null | grep -qw data && echo "yes" || echo "no")
if [ "$VOLUME_EXISTS" = "yes" ]; then
  echo "✅ 볼륨 존재 (data)"
else
  echo "❌ 볼륨 없음 (data 볼륨이 없습니다)"
fi

# 6. Sidecar Volume Mount 확인
echo ""
echo "=== Sidecar Volume Mount 확인 ==="
SIDECAR_MOUNT=$(kubectl get deploy wordpress -o jsonpath='{.spec.template.spec.containers[?(@.name=="sidecar")].volumeMounts[?(@.mountPath=="/var/log")].name}' 2>/dev/null)
if [ "$SIDECAR_MOUNT" = "data" ]; then
  echo "✅ sidecar 마운트 OK (/var/log -> data)"
else
  echo "❌ sidecar 마운트 없음 (현재: ${SIDECAR_MOUNT:-없음})"
fi

# 7. WordPress Volume Mount 확인
echo ""
echo "=== WordPress Volume Mount 확인 ==="
WP_MOUNT=$(kubectl get deploy wordpress -o jsonpath='{.spec.template.spec.containers[?(@.name=="wordpress")].volumeMounts[?(@.mountPath=="/var/log")].name}' 2>/dev/null)
if [ "$WP_MOUNT" = "data" ]; then
  echo "✅ wordpress 마운트 OK (/var/log -> data)"
else
  echo "❌ wordpress 마운트 없음 (현재: ${WP_MOUNT:-없음})"
fi

# 8. Pod 상태 및 로그 확인
echo ""
echo "=== Pod 상태 및 로그 확인 ==="
POD_NAME=$(kubectl get pods -l app=wordpress -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)
if [ -n "$POD_NAME" ]; then
  POD_STATUS=$(kubectl get pod $POD_NAME -o jsonpath='{.status.phase}' 2>/dev/null)
  echo "Pod 이름: $POD_NAME"
  echo "Pod 상태: $POD_STATUS"
  
  if [ "$POD_STATUS" = "Running" ]; then
    # Sidecar 컨테이너가 실행 중인지 확인
    SIDECAR_READY=$(kubectl get pod $POD_NAME -o jsonpath='{.status.containerStatuses[?(@.name=="sidecar")].ready}' 2>/dev/null)
    if [ "$SIDECAR_READY" = "true" ]; then
      echo "✅ Sidecar 컨테이너 실행 중"
      
      # Sidecar 로그 확인 (최근 10줄)
      echo ""
      echo "=== Sidecar 로그 확인 (최근 10줄) ==="
      SIDECAR_LOGS=$(kubectl logs $POD_NAME -c sidecar --tail=10 2>/dev/null)
      if [ -n "$SIDECAR_LOGS" ]; then
        echo "✅ Sidecar 로그 출력 중:"
        echo "$SIDECAR_LOGS"
        # tail 명령어가 실행 중인지 확인 (로그가 계속 출력되는지)
        if echo "$SIDECAR_LOGS" | grep -q "tail"; then
          echo "✅ tail 명령어가 실행 중입니다"
        else
          echo "⚠️  tail 명령어 관련 로그가 보이지 않습니다"
        fi
      else
        echo "⚠️  Sidecar 로그가 없습니다 (컨테이너가 방금 시작되었을 수 있음)"
      fi
    else
      echo "❌ Sidecar 컨테이너가 Ready 상태가 아닙니다"
      echo "   Sidecar 컨테이너 상태:"
      kubectl get pod $POD_NAME -o jsonpath='{.status.containerStatuses[?(@.name=="sidecar")]}' 2>/dev/null | echo
    fi
  else
    echo "⚠️  Pod가 Running 상태가 아닙니다 (상태: $POD_STATUS)"
  fi
else
  echo "❌ Pod를 찾을 수 없습니다"
fi

# 9. 최종 요약
echo ""
echo "=== 최종 요약 ==="
# 명령어 확인을 위한 변수 재정의
FULL_CMD_CHECK=$(echo "$SIDECAR_COMMAND" | tr ' ' ' ')
if [ "$SIDECAR_EXISTS" = "yes" ] && \
   [ "$SIDECAR_IMAGE" = "busybox:stable" ] && \
   [ -n "$SIDECAR_COMMAND" ] && \
   echo "$FULL_CMD_CHECK" | grep -qE "sh.*-c.*tail -[fF].*wordpress.log" && \
   [ "$VOLUME_EXISTS" = "yes" ] && \
   [ "$SIDECAR_MOUNT" = "data" ] && \
   [ "$WP_MOUNT" = "data" ]; then
  echo "✅✅✅ 모든 요구사항 충족"
else
  echo "❌❌❌ 일부 요구사항 미충족"
fi
