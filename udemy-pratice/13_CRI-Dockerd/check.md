echo "=== 정답확인 (QUESTION 13) ==="

# cri-dockerd 패키지 설치 확인
echo ""
echo "=== cri-dockerd 패키지 설치 확인 ==="
if dpkg -l | grep -q cri-dockerd; then
  echo "✅ cri-dockerd 패키지 설치됨"
  dpkg -l | grep cri-dockerd
else
  echo "❌ cri-dockerd 패키지가 설치되지 않았습니다"
fi

# cri-docker 서비스 상태 확인
echo ""
echo "=== cri-docker 서비스 상태 확인 ==="
if systemctl is-enabled cri-docker >/dev/null 2>&1; then
  ENABLED_STATUS=$(systemctl is-enabled cri-docker)
  if [ "$ENABLED_STATUS" = "enabled" ]; then
    echo "✅ cri-docker 서비스가 활성화됨 (enabled)"
  else
    echo "⚠️  cri-docker 서비스 활성화 상태: $ENABLED_STATUS"
  fi
else
  echo "❌ cri-docker 서비스가 설치되지 않았거나 활성화되지 않았습니다"
fi

# cri-docker 서비스 실행 상태 확인
echo ""
echo "=== cri-docker 서비스 실행 상태 확인 ==="
if systemctl is-active cri-docker >/dev/null 2>&1; then
  ACTIVE_STATUS=$(systemctl is-active cri-docker)
  if [ "$ACTIVE_STATUS" = "active" ]; then
    echo "✅ cri-docker 서비스가 실행 중 (active)"
  else
    echo "⚠️  cri-docker 서비스 실행 상태: $ACTIVE_STATUS"
  fi
else
  echo "❌ cri-docker 서비스가 실행되지 않았습니다"
fi

# 시스템 파라미터 확인
echo ""
echo "=== 시스템 파라미터 확인 ==="

# net.bridge.bridge-nf-call-iptables
BRIDGE_NF_IPTABLES=$(sysctl -n net.bridge.bridge-nf-call-iptables 2>/dev/null || echo "확인 불가")
if [ "$BRIDGE_NF_IPTABLES" = "1" ]; then
  echo "✅ net.bridge.bridge-nf-call-iptables = 1"
else
  echo "❌ net.bridge.bridge-nf-call-iptables = ${BRIDGE_NF_IPTABLES} (기대값: 1)"
fi

# net.ipv6.conf.all.forwarding
IPV6_FORWARDING=$(sysctl -n net.ipv6.conf.all.forwarding 2>/dev/null || echo "확인 불가")
if [ "$IPV6_FORWARDING" = "1" ]; then
  echo "✅ net.ipv6.conf.all.forwarding = 1"
else
  echo "❌ net.ipv6.conf.all.forwarding = ${IPV6_FORWARDING} (기대값: 1)"
fi

# net.ipv4.ip_forward
IPV4_FORWARD=$(sysctl -n net.ipv4.ip_forward 2>/dev/null || echo "확인 불가")
if [ "$IPV4_FORWARD" = "1" ]; then
  echo "✅ net.ipv4.ip_forward = 1"
else
  echo "❌ net.ipv4.ip_forward = ${IPV4_FORWARD} (기대값: 1)"
fi

# net.netfilter.nf_conntrack_max
NF_CONNTRACK_MAX=$(sysctl -n net.netfilter.nf_conntrack_max 2>/dev/null || echo "확인 불가")
if [ "$NF_CONNTRACK_MAX" = "131072" ]; then
  echo "✅ net.netfilter.nf_conntrack_max = 131072"
else
  echo "❌ net.netfilter.nf_conntrack_max = ${NF_CONNTRACK_MAX} (기대값: 131072)"
fi

# 서비스 상세 정보
echo ""
echo "=== cri-docker 서비스 상세 정보 ==="
systemctl status cri-docker --no-pager -l 2>/dev/null || echo "서비스 정보를 가져올 수 없습니다"

# 모든 시스템 파라미터 확인
echo ""
echo "=== 모든 시스템 파라미터 확인 ==="
echo "net.bridge.bridge-nf-call-iptables: $(sysctl -n net.bridge.bridge-nf-call-iptables 2>/dev/null || echo 'N/A')"
echo "net.ipv6.conf.all.forwarding: $(sysctl -n net.ipv6.conf.all.forwarding 2>/dev/null || echo 'N/A')"
echo "net.ipv4.ip_forward: $(sysctl -n net.ipv4.ip_forward 2>/dev/null || echo 'N/A')"
echo "net.netfilter.nf_conntrack_max: $(sysctl -n net.netfilter.nf_conntrack_max 2>/dev/null || echo 'N/A')"

# 요구사항 요약
echo ""
echo "=== 요구사항 확인 요약 ==="
PACKAGE_INSTALLED=$(dpkg -l | grep -q cri-dockerd && echo "✅" || echo "❌")
SERVICE_ENABLED=$(systemctl is-enabled cri-docker 2>/dev/null | grep -q enabled && echo "✅" || echo "❌")
SERVICE_ACTIVE=$(systemctl is-active cri-docker 2>/dev/null | grep -q active && echo "✅" || echo "❌")
SYSCTL1=$(if [ "$BRIDGE_NF_IPTABLES" = "1" ]; then echo "✅"; else echo "❌"; fi)
SYSCTL2=$(if [ "$IPV6_FORWARDING" = "1" ]; then echo "✅"; else echo "❌"; fi)
SYSCTL3=$(if [ "$IPV4_FORWARD" = "1" ]; then echo "✅"; else echo "❌"; fi)
SYSCTL4=$(if [ "$NF_CONNTRACK_MAX" = "131072" ]; then echo "✅"; else echo "❌"; fi)

echo "1. 패키지 설치: $PACKAGE_INSTALLED"
echo "2. 서비스 활성화: $SERVICE_ENABLED"
echo "3. 서비스 시작: $SERVICE_ACTIVE"
echo "4. net.bridge.bridge-nf-call-iptables=1: $SYSCTL1"
echo "5. net.ipv6.conf.all.forwarding=1: $SYSCTL2"
echo "6. net.ipv4.ip_forward=1: $SYSCTL3"
echo "7. net.netfilter.nf_conntrack_max=131072: $SYSCTL4"
