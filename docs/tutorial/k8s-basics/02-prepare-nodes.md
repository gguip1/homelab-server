# 2. 노드 공통 설정

컨트롤 플레인과 워커 세 대 모두에서 같은 순서로 진행, 새 실습 VM 기준

## 1. VM 접속

**Mac에서**, IP를 바꾸어 각 VM에 접속

```sh
ssh -J pve ubuntu@10.10.10.110
```

이하 명령은 **각 VM의 ubuntu 계정에서** 실행, Proxmox 호스트에서 실행 금지

## 2. 패키지와 swap 설정

```sh
sudo apt-get update
sudo apt-get install -y ca-certificates curl gpg containerd
sudo swapoff -a
sudo cp -n /etc/fstab /etc/fstab.before-k8s
sudo sed -i '/^[[:space:]]*#/! { /[[:space:]]swap[[:space:]]/s/^/# /; }' /etc/fstab
swapon --show
systemctl list-units --type=swap --all
```

확인: 활성 swap 없음, `/etc/fstab`의 swap 항목 주석 처리

별도 swap·zram 서비스가 있으면 재부팅 후 재활성화되지 않도록 해당 서비스 설정 확인 필요

## 3. 커널 모듈과 패킷 전달

```sh
cat <<'EOF' | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

cat <<'EOF' | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
sudo sysctl --system
```

확인

```sh
lsmod | grep -E 'overlay|br_netfilter'
sysctl net.ipv4.ip_forward net.bridge.bridge-nf-call-iptables
```

`ip_forward`와 `bridge-nf-call-iptables` 모두 `1` 확인

## 4. containerd 설정

새 VM의 기본 설정을 생성하는 단계, 기존 서비스용 containerd 설정에 덮어쓰기 금지

```sh
sudo install -d -m 755 /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
sudo systemctl enable --now containerd
sudo systemctl restart containerd
containerd --version
sudo containerd config dump | grep -E 'SystemdCgroup|disabled_plugins'
systemctl is-active containerd
```

확인: `SystemdCgroup = true`, `disabled_plugins`에 `cri` 없음, 서비스 `active`

kubelet과 런타임의 cgroup 드라이버를 systemd로 맞추는 설정, 위 출력이 없거나 다르면 진행 중단 후 [런타임 공식 문서](https://v1-35.docs.kubernetes.io/docs/setup/production-environment/container-runtimes/)와 버전별 설정 확인

## 5. Kubernetes 패키지 저장소

```sh
sudo install -d -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key -o /tmp/kubernetes-release.key
sudo gpg --dearmor --yes -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg /tmp/kubernetes-release.key
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
apt-cache madison kubeadm
```

확인: v1.35 계열 패키지 목록, `curl`·`gpg`·`apt-get update` 오류가 있으면 다음 단계 진행 금지

## 6. 같은 패치 버전 설치

먼저 컨트롤 플레인에서 실행

```sh
K8S_PACKAGE_VERSION=$(apt-cache madison kubeadm | awk 'NR==1 {print $3}')
printf '%s\n' "$K8S_PACKAGE_VERSION"
```

출력된 전체 패키지 버전 문자열을 별도로 기록, 빈 값이면 저장소 설정 재확인

워커 두 대에서는 아래 값에 컨트롤 플레인에서 기록한 문자열을 직접 입력

```sh
K8S_PACKAGE_VERSION='<컨트롤-플레인과-동일한-패키지-버전>'
```

**세 노드 모두에서**, 변수가 올바르게 설정된 같은 터미널에서 실행

```sh
sudo apt-get install -y kubelet="$K8S_PACKAGE_VERSION" kubeadm="$K8S_PACKAGE_VERSION" kubectl="$K8S_PACKAGE_VERSION"
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable kubelet
kubeadm version -o short
kubectl version --client
dpkg-query -W kubeadm kubelet kubectl containerd
ls /opt/cni/bin/bridge /opt/cni/bin/host-local /opt/cni/bin/loopback /opt/cni/bin/portmap
```

확인: 세 노드의 kubeadm·kubelet·kubectl 버전 동일, 기본 CNI 실행 파일 존재

클러스터 생성 전 kubelet이 재시작을 반복하는 것은 설정 파일 생성 전 상태일 수 있음, containerd는 정상 동작 필요

패키지 hold는 의도하지 않은 버전 변경 방지용, 보안 업데이트와 버전 업그레이드는 이후 별도 절차 필요

## 7. 재부팅 후 확인

```sh
sudo reboot
```

Mac에서 같은 VM에 다시 접속 후 실행

```sh
hostname
swapon --show
sysctl net.ipv4.ip_forward net.bridge.bridge-nf-call-iptables
systemctl is-active containerd
timedatectl status
```

확인: swap 없음, sysctl 값 유지, containerd `active`, 시스템 시각 동기화

세 노드 모두 완료 후 다음 장 진행

다음: [클러스터 생성](03-create-cluster.md)

참고: [kubeadm 설치 준비](https://v1-35.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
