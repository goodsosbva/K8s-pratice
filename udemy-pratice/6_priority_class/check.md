echo "=== 정답확인 (QUESTION 06) ==="

# Priority Class 존재 확인
kubectl get priorityclass high-priority >/dev/null 2>&1 && echo "✅ Priority Class 존재" || echo "❌ Priority Class 없음"

# Priority Class 값 확인 (기존 최고값 - 1이어야 함)
# high-priority를 제외한 모든 Priority Class 중 최고값 찾기
HIGHEST_VALUE=$(kubectl get priorityclass -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.value}{"\n"}{end}' | grep -v "^high-priority" | grep -v "^system-" | awk '{print $2}' | sort -n | tail -1)

if [ -z "$HIGHEST_VALUE" ]; then
  echo "❌ 기존 Priority Class를 찾을 수 없습니다"
else
  EXPECTED_VALUE=$((HIGHEST_VALUE - 1))
  ACTUAL_VALUE=$(kubectl get priorityclass high-priority -o jsonpath='{.value}')
  
  if [ "$ACTUAL_VALUE" -eq "$EXPECTED_VALUE" ]; then
    echo "✅ Priority Class 값 올바름 (기대값: $EXPECTED_VALUE, 실제값: $ACTUAL_VALUE)"
  else
    echo "❌ Priority Class 값 불일치 (기대값: $EXPECTED_VALUE, 실제값: $ACTUAL_VALUE)"
  fi
fi

# Deployment가 Priority Class를 사용하는지 확인
PRIORITY_CLASS=$(kubectl get deployment busybox-logger -n priority -o jsonpath='{.spec.template.spec.priorityClassName}')

if [ "$PRIORITY_CLASS" = "high-priority" ]; then
  echo "✅ Deployment가 high-priority Priority Class 사용"
else
  echo "❌ Deployment가 올바른 Priority Class를 사용하지 않음 (현재: ${PRIORITY_CLASS:-없음})"
fi

# Pod가 Priority Class를 사용하는지 확인
POD_PRIORITY_CLASS=$(kubectl get pod -n priority -l app=busybox-logger -o jsonpath='{.items[0].spec.priorityClassName}' 2>/dev/null)

if [ "$POD_PRIORITY_CLASS" = "high-priority" ]; then
  echo "✅ Pod가 high-priority Priority Class 사용"
else
  echo "❌ Pod가 올바른 Priority Class를 사용하지 않음 (현재: ${POD_PRIORITY_CLASS:-없음})"
fi
