# 1. 파일 존재 확인

# 2. 파일 내용 확인

# 3. CRD 제외 확인

# 4. 네임스페이스 확인

[ -f /home/argo/argo-helm.yaml ] && echo "✅ 파일 생성 성공" || echo "❌ 파일 생성 실패"

[ -s /home/argo/argo-helm.yaml ] && echo "✅ 파일에 내용 있음" || echo "❌ 파일 비어있음"

grep -qi "CustomResourceDefinition" /home/argo/argo-helm.yaml && echo "❌ CRD 포함됨" || echo "✅ CRD 제외됨"

grep -q "namespace: argocd" /home/argo/argo-helm.yaml && echo "✅ 네임스페이스 설정됨" || echo "❌ 네임스페이스 없음"

echo 정답확인

# 5. 한 번에 확인 (가장 간단)

[ -f /home/argo/argo-helm.yaml ] && [ -s /home/argo/argo-helm.yaml ] && ! grep -qi "CustomResourceDefinition" /home/argo/argo-helm.yaml && grep -q "namespace: argocd" /home/argo/argo-helm.yaml && echo "✅✅✅ 성공" || echo "❌❌❌ 실패"
