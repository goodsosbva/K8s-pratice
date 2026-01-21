echo "=== 정답확인 (QUESTION 05) ==="

kubectl get storageclass local-kiddie >/dev/null 2>&1 && echo "✅ StorageClass 존재" || echo "❌ StorageClass 없음"

kubectl get storageclass local-kiddie -o jsonpath='{.provisioner}' | grep -qx 'rancher.io/local-path' && echo "✅ Provisioner OK" || echo "❌ Provisioner 불일치"

kubectl get storageclass local-kiddie -o jsonpath='{.volumeBindingMode}' | grep -qx 'WaitForFirstConsumer' && echo "✅ volumeBindingMode OK" || echo "❌ volumeBindingMode 불일치"

kubectl get storageclass local-kiddie -o jsonpath='{.metadata.annotations.storageclass\.kubernetes\.io/is-default-class}' | grep -qx 'true' && echo "✅ 기본 StorageClass 설정됨" || echo "❌ 기본 StorageClass 설정 안 됨"

kubectl get storageclass | grep local-kiddie | grep -q "(default)" && echo "✅ 기본 StorageClass 표시됨" || echo "❌ 기본 StorageClass 표시 안 됨"
