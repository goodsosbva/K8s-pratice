echo "=== 정답확인 (QUESTION 02) ==="

kubectl get deploy wordpress >/dev/null 2>&1 && echo "✅ deployment 존재" || echo "❌ deployment 없음"

kubectl get deploy wordpress -o jsonpath='{.spec.template.spec.initContainers[*].name} {.spec.template.spec.containers[*].name}' | grep -qw sidecar && echo "✅ sidecar 존재" || echo "❌ sidecar 없음"

kubectl get deploy wordpress -o jsonpath='{.spec.template.spec.initContainers[?(@.name=="sidecar")].image} {.spec.template.spec.containers[?(@.name=="sidecar")].image}' | grep -qw 'busybox:stable' && echo "✅ 이미지 OK" || echo "❌ 이미지 불일치"

kubectl get deploy wordpress -o jsonpath='{.spec.template.spec.initContainers[?(@.name=="sidecar")].command} {.spec.template.spec.containers[?(@.name=="sidecar")].command}' | grep -q 'tail -f /var/log/wordpress.log' && echo "✅ 커맨드 OK" || echo "❌ 커맨드 불일치"

kubectl get deploy wordpress -o jsonpath='{.spec.template.spec.volumes[*].name}' | grep -qw data && echo "✅ 볼륨 존재" || echo "❌ 볼륨 없음"

kubectl get deploy wordpress -o jsonpath='{.spec.template.spec.initContainers[?(@.name=="sidecar")].volumeMounts[?(@.mountPath=="/var/log")].name} {.spec.template.spec.containers[?(@.name=="sidecar")].volumeMounts[?(@.mountPath=="/var/log")].name}' | grep -qw data && echo "✅ sidecar 마운트 OK" || echo "❌ sidecar 마운트 없음"

kubectl get deploy wordpress -o jsonpath='{.spec.template.spec.containers[?(@.name!="sidecar")].volumeMounts[?(@.mountPath=="/var/log")].name}' | grep -qw data && echo "✅ wordpress 마운트 OK" || echo "❌ wordpress 마운트 없음"
