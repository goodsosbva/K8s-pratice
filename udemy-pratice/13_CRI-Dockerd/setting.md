# 실습 환경 설정 (CRI-Dockerd)

#!/bin/bash
set -e

echo "=========================================="
echo "  CRI-Dockerd 설치 및 설정 스크립트"
echo "=========================================="

# 1. 필수 커널 모듈 로드
echo ""
echo "=== 1. 필수 커널 모듈 로드 ==="
sudo modprobe overlay
sudo modprobe br_netfilter
echo "✅ overlay, br_netfilter 모듈 로드 완료"

# 모듈 영구 설정
cat <<EOF | sudo tee /etc/modules-load.d/cri-docker.conf
overlay
br_netfilter
EOF
echo "✅ 커널 모듈 영구 설정 완료"

# 2. 시스템 파라미터 설정 (먼저 설정해야 함)
echo ""
echo "=== 2. 시스템 파라미터 설정 ==="
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
net.ipv6.conf.all.forwarding        = 1
EOF
sudo sysctl --system
echo "✅ 시스템 파라미터 설정 완료"

# 3. Docker 서비스 확인 및 시작
echo ""
echo "=== 3. Docker 서비스 확인 ==="
if ! systemctl is-active --quiet docker; then
  echo "⚠️  Docker가 실행 중이 아닙니다. 시작합니다..."
  sudo systemctl start docker
  sudo systemctl enable docker
fi
echo "✅ Docker 서비스 실행 중"

# 4. docker 그룹 확인 및 생성
echo ""
echo "=== 4. docker 그룹 확인 ==="
if getent group docker > /dev/null 2>&1; then
  echo "✅ docker 그룹 존재"
else
  echo "⚠️  docker 그룹 생성 중..."
  sudo groupadd docker
  echo "✅ docker 그룹 생성 완료"
fi

# 5. cri-dockerd 패키지 다운로드 및 설치
echo ""
echo "=== 5. cri-dockerd 패키지 설치 ==="
CRI_DOCKERD_VERSION="0.3.9"
CRI_DOCKERD_DEB="cri-dockerd_${CRI_DOCKERD_VERSION}.3-0.ubuntu-jammy_amd64.deb"

if ! dpkg -l | grep -q cri-dockerd; then
  if [ ! -f ~/${CRI_DOCKERD_DEB} ]; then
    echo "패키지 다운로드 중..."
    wget -q https://github.com/Mirantis/cri-dockerd/releases/download/v${CRI_DOCKERD_VERSION}/${CRI_DOCKERD_DEB} -O ~/${CRI_DOCKERD_DEB}
  fi
  echo "패키지 설치 중..."
  sudo dpkg -i ~/${CRI_DOCKERD_DEB}
  echo "✅ cri-dockerd 패키지 설치 완료"
else
  echo "✅ cri-dockerd 패키지가 이미 설치됨"
fi

# 6. systemd 재로드
echo ""
echo "=== 6. systemd 재로드 ==="
sudo systemctl daemon-reload
echo "✅ systemd daemon-reload 완료"

# 7. cri-docker 서비스 활성화 및 시작
echo ""
echo "=== 7. cri-docker 서비스 시작 ==="
sudo systemctl enable cri-docker.socket
sudo systemctl enable cri-docker.service
sudo systemctl start cri-docker.socket
sudo systemctl start cri-docker.service
echo "✅ cri-docker 서비스 시작 완료"

# 8. 상태 확인
echo ""
echo "=== 8. 최종 상태 확인 ==="
echo ""
echo "--- Docker 상태 ---"
systemctl is-active docker && echo "✅ Docker: 실행 중" || echo "❌ Docker: 실행 안됨"
echo ""
echo "--- cri-docker 상태 ---"
systemctl is-active cri-docker && echo "✅ cri-docker: 실행 중" || echo "❌ cri-docker: 실행 안됨"
echo ""
echo "--- 소켓 파일 확인 ---"
ls -la /var/run/cri-dockerd.sock 2>/dev/null && echo "✅ 소켓 파일 존재" || echo "❌ 소켓 파일 없음"
echo ""
echo "=========================================="
echo "  설정 완료!"
echo "=========================================="
