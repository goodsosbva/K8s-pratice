# 실습 환경 설정

# cert-manager 설치 확인
echo "=== cert-manager 설치 확인 ==="
if kubectl get crds | grep -q cert-manager; then
  echo "✅ cert-manager가 이미 설치되어 있습니다"
else
  echo "⚠️  cert-manager가 설치되어 있지 않습니다. 설치를 시작합니다..."
  
  # cert-manager 설치
  kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml
  
  echo "cert-manager 설치 완료. CRD가 생성될 때까지 대기 중..."
  sleep 10
  
  # CRD 생성 확인
  kubectl wait --for condition=established --timeout=60s crd/certificates.cert-manager.io 2>/dev/null || echo "CRD 생성 대기 중..."
fi

# cert-manager CRD 목록 확인
echo "=== cert-manager CRD 목록 ==="
kubectl get crds | grep cert-manager

# Certificate CRD 확인
echo ""
echo "=== Certificate CRD 상세 정보 ==="
kubectl get crd certificates.cert-manager.io -o yaml

# subject 필드 확인
echo ""
echo "=== Certificate CRD의 subject 필드 경로 ==="
if kubectl get crd certificates.cert-manager.io >/dev/null 2>&1; then
  kubectl explain certificate.spec.subject 2>/dev/null || \
  kubectl explain certificates.cert-manager.io.spec.subject 2>/dev/null || \
  echo "explain 명령어로 확인하려면: kubectl explain certificate.spec.subject"
else
  echo "⚠️  Certificate CRD가 아직 생성되지 않았습니다"
fi

# 해결 방법 예시 (주석)
# Task 1: cert-manager CRD 리스트 저장
# kubectl get crds | grep cert-manager > ~/resources.yaml
# 또는
# kubectl get crds -o yaml | grep -A 5 -B 5 cert-manager > ~/resources.yaml

# Task 2: Certificate의 subject 필드 문서 추출
# kubectl explain certificate.spec.subject > ~/subject.yaml
# 또는
# kubectl explain crd certificates.cert-manager.io --recursive | grep -A 20 "subject" > ~/subject.yaml
