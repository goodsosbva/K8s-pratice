# 실습 환경 설정

# docker 그룹 확인 및 생성 (cri-dockerd가 필요로 함)
echo "=== docker 그룹 확인 ==="
if getent group docker > /dev/null 2>&1; then
  echo "✅ docker 그룹 존재"
else
  echo "⚠️  docker 그룹이 없습니다. 생성 중..."
  sudo groupadd docker
  echo "✅ docker 그룹 생성 완료"
fi

# cri-dockerd 패키지 파일 확인 및 다운로드
echo ""
echo "=== cri-dockerd 패키지 파일 확인 ==="
if [ -f ~/cri-dockerd_0.3.9.3-0.ubuntu-jammy_amd64.deb ]; then
  echo "✅ 패키지 파일 존재"
  ls -lh ~/cri-dockerd_0.3.9.3-0.ubuntu-jammy_amd64.deb
else
  echo "⚠️  패키지 파일이 없습니다. 다운로드 시도 중..."
  wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.9/cri-dockerd_0.3.9.3-0.ubuntu-jammy_amd64.deb -O ~/cri-dockerd_0.3.9.3-0.ubuntu-jammy_amd64.deb
  if [ $? -eq 0 ] && [ -f ~/cri-dockerd_0.3.9.3-0.ubuntu-jammy_amd64.deb ]; then
    echo "✅ 패키지 파일 다운로드 완료"
    ls -lh ~/cri-dockerd_0.3.9.3-0.ubuntu-jammy_amd64.deb
  else
    echo "❌ 다운로드 실패. 수동으로 파일을 준비하세요."
    echo "   URL: https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.9/cri-dockerd_0.3.9.3-0.ubuntu-jammy_amd64.deb"
  fi
fi

# 패키지 설치 여부 확인
echo ""
echo "=== cri-dockerd 패키지 설치 확인 ==="
if dpkg -l | grep -q cri-dockerd; then
  echo "✅ cri-dockerd 패키지가 이미 설치되어 있습니다"
  dpkg -l | grep cri-dockerd
  echo ""
  echo "⚠️  패키지가 이미 설치되어 있으면 systemctl daemon-reload를 실행하세요:"
  echo "   sudo systemctl daemon-reload"
else
  echo "ℹ️  cri-dockerd 패키지가 설치되지 않았습니다"
  echo "   설치 후 systemctl daemon-reload를 실행하세요:"
  echo "   sudo dpkg -i ~/cri-dockerd_0.3.9.3-0.ubuntu-jammy_amd64.deb"
  echo "   sudo systemctl daemon-reload"
fi

# 현재 시스템 파라미터 확인
echo ""
echo "=== 현재 시스템 파라미터 확인 ==="
echo "net.bridge.bridge-nf-call-iptables: $(sysctl -n net.bridge.bridge-nf-call-iptables 2>/dev/null || echo '확인 불가')"
echo "net.ipv6.conf.all.forwarding: $(sysctl -n net.ipv6.conf.all.forwarding 2>/dev/null || echo '확인 불가')"
echo "net.ipv4.ip_forward: $(sysctl -n net.ipv4.ip_forward 2>/dev/null || echo '확인 불가')"
echo "net.netfilter.nf_conntrack_max: $(sysctl -n net.netfilter.nf_conntrack_max 2>/dev/null || echo '확인 불가')"

# cri-docker 서비스 상태 확인
echo ""
echo "=== cri-docker 서비스 상태 확인 ==="
systemctl status cri-docker --no-pager 2>/dev/null || echo "cri-docker 서비스가 아직 설치되지 않았습니다"

# 설치 방법 안내
echo ""
echo "=== 설치 방법 ==="
echo "1. 패키지 설치:"
echo "   sudo dpkg -i ~/cri-dockerd_0.3.9.3-0.ubuntu-jammy_amd64.deb"
echo ""
echo "2. systemd 재로드 (패키지 설치 후 필수):"
echo "   sudo systemctl daemon-reload"
echo ""
echo "3. 서비스 활성화 및 시작:"
echo "   sudo systemctl enable cri-docker"
echo "   sudo systemctl start cri-docker"
echo ""
echo "4. 시스템 파라미터 설정:"
echo "   sudo sysctl -w net.bridge.bridge-nf-call-iptables=1"
echo "   sudo sysctl -w net.ipv6.conf.all.forwarding=1"
echo "   sudo sysctl -w net.ipv4.ip_forward=1"
echo "   sudo sysctl -w net.netfilter.nf_conntrack_max=131072"
echo ""
echo "5. 영구 설정 (선택사항):"
echo "   /etc/sysctl.d/cka.conf 파일에 추가하여 재부팅 후에도 유지"
