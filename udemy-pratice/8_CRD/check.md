echo "=== 정답확인 (QUESTION 08) ==="

# Task 1: cert-manager CRD 리스트 확인 (~/resources.yaml)
echo ""
echo "=== Task 1: cert-manager CRD 리스트 ==="
if [ -f ~/resources.yaml ]; then
  echo "✅ resources.yaml 파일 존재"
  
  # 파일 내용 확인 (cert-manager CRD가 포함되어 있는지)
  CERT_MANAGER_COUNT=$(grep -i "cert-manager" ~/resources.yaml | wc -l || echo "0")
  if [ "$CERT_MANAGER_COUNT" -gt 0 ]; then
    echo "✅ cert-manager CRD 포함됨"
  else
    echo "⚠️  cert-manager CRD가 포함되지 않았을 수 있음"
  fi
  
  # YAML 형식 확인
  if grep -q "^apiVersion:" ~/resources.yaml || grep -q "^kind:" ~/resources.yaml || grep -q "NAME" ~/resources.yaml; then
    echo "✅ YAML 또는 kubectl 기본 형식 확인됨"
  else
    echo "⚠️  파일 형식 확인 필요"
  fi
  
  # 파일 내용 미리보기
  echo ""
  echo "--- resources.yaml 내용 (처음 10줄) ---"
  head -10 ~/resources.yaml
else
  echo "❌ resources.yaml 파일 없음"
fi

# Task 2: Certificate의 subject 필드 문서 확인 (~/subject.yaml)
echo ""
echo "=== Task 2: Certificate subject 필드 문서 ==="
if [ -f ~/subject.yaml ]; then
  echo "✅ subject.yaml 파일 존재"
  
  # subject 필드 관련 내용 확인
  if grep -qi "subject" ~/subject.yaml; then
    echo "✅ subject 필드 관련 내용 포함됨"
  else
    echo "⚠️  subject 필드 관련 내용이 없을 수 있음"
  fi
  
  # 파일 내용 미리보기
  echo ""
  echo "--- subject.yaml 내용 ---"
  cat ~/subject.yaml
else
  echo "❌ subject.yaml 파일 없음"
fi

# 추가 확인: 실제 cert-manager CRD 목록과 비교
echo ""
echo "=== 추가 확인: 현재 cert-manager CRD 목록 ==="
CURRENT_CRDS=$(kubectl get crds | grep cert-manager | wc -l)
echo "현재 클러스터의 cert-manager CRD 개수: $CURRENT_CRDS"

if [ "$CURRENT_CRDS" -gt 0 ]; then
  echo "✅ cert-manager CRD가 클러스터에 설치되어 있음"
  echo ""
  echo "cert-manager CRD 목록:"
  kubectl get crds | grep cert-manager
else
  echo "⚠️  cert-manager CRD가 클러스터에 설치되어 있지 않음"
  echo "   세팅 스크립트를 실행하여 cert-manager를 설치하세요"
fi
