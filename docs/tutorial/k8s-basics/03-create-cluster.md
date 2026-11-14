# 3. 클러스터 생성

시작: [노드 공통 설정](02-prepare-nodes.md)을 세 VM 모두에서 완료

## 1. 컨트롤 플레인 접속

**Mac에서**

```sh
ssh -J pve ubuntu@10.10.10.110
```

이하 4단계까지 **컨트롤 플레인에서만** 실행

## 2. 초기화

```sh
K8S_VERSION=$(kubeadm version -o short)
sudo kubeadm init \
  --kubernetes-version "$K8S_VERSION" \
  --apiserver-advertise-address=10.10.10.110 \
  --pod-network-cidr=10.244.0.0/16 \
  --service-cidr=10.96.0.0/12 \
  --cri-socket=unix:///run/containerd/containerd.sock
```

| 옵션 | 의미 |
|---|---|
| kubernetes-version | 설치한 kubeadm과 같은 Kubernetes 버전 사용 |
| apiserver-advertise-address | 이 컨트롤 플레인의 API 주소 |
| pod-network-cidr | Flannel 기본값에 맞춘 Pod 주소 범위 |
| service-cidr | ClusterIP용 가상 주소 범위 |
| cri-socket | containerd의 CRI 소켓 |

확인: `Your Kubernetes control-plane has initialized successfully!`

출력의 `kubeadm join ...`은 워커 가입용 명령, 토큰 포함으로 Git·스크린샷·공개 로그에 저장 금지

초기화 실패 시 오류부터 확인, 성공한 노드에서 같은 명령을 반복 실행하지 않음

## 3. ubuntu 계정의 kubectl 설정

```sh
mkdir -p "$HOME/.kube"
sudo install -m 600 -o "$(id -u)" -g "$(id -g)" /etc/kubernetes/admin.conf "$HOME/.kube/config"
kubectl get nodes -o wide
kubectl get pods -n kube-system
```

확인: 컨트롤 플레인 1대 표시, CNI 설치 전에는 `NotReady`와 CoreDNS `Pending` 가능

`admin.conf`와 `.kube/config`는 관리자 자격증명 포함, 저장소에 복사 금지

## 4. Flannel 설치

```sh
curl -fL https://github.com/flannel-io/flannel/releases/download/v0.28.9/kube-flannel.yml -o /tmp/kube-flannel-v0.28.9.yml
grep -nE 'image:|Network|Backend|privileged:|hostNetwork:|NET_ADMIN|NET_RAW' /tmp/kube-flannel-v0.28.9.yml
```

확인: Flannel 버전, Pod 네트워크 `10.244.0.0/16`, VXLAN backend, `hostNetwork: true`, `NET_ADMIN`·`NET_RAW` 권한 사용

다운로드 성공과 내용 확인 후 적용

```sh
kubectl apply -f /tmp/kube-flannel-v0.28.9.yml
kubectl -n kube-flannel rollout status daemonset/kube-flannel-ds --timeout=180s
kubectl -n kube-system rollout status deployment/coredns --timeout=180s
kubectl get nodes -o wide
```

확인: 컨트롤 플레인 `Ready`, Flannel과 CoreDNS 정상 실행

Flannel은 Pod 간 연결용 구성, 이 구성만으로 NetworkPolicy 접근 제어 제공 없음

참고: [Flannel v0.28.9 설치 안내](https://github.com/flannel-io/flannel/blob/v0.28.9/README.md)

## 5. 워커 가입

**컨트롤 플레인에서** 가입 명령 생성

```sh
sudo kubeadm token create --ttl 30m --print-join-command
```

출력된 실제 명령을 복사, `--discovery-token-ca-cert-hash` 포함 여부 확인

**Mac의 다른 터미널에서** 첫 워커 접속

```sh
ssh -J pve ubuntu@10.10.10.111
```

**워커에서** 출력받은 명령 앞에 `sudo`, 뒤에 CRI 소켓 옵션을 붙여 실행하는 형태

```text
sudo kubeadm join 10.10.10.110:6443 --token <실제-토큰> --discovery-token-ca-cert-hash sha256:<실제-해시> --cri-socket=unix:///run/containerd/containerd.sock
```

확인: `This node has joined the cluster`

두 번째 워커 `10.10.10.112`에서도 같은 방식으로 실행, `kubeadm init`은 워커에서 실행하지 않음

토큰 만료 시 컨트롤 플레인에서 새로 생성, 인증 검증을 건너뛰는 옵션 사용 불필요

## 6. 클러스터 확인

**컨트롤 플레인에서**

```sh
kubectl wait --for=condition=Ready node/k8s-lab-cp01 node/k8s-lab-w01 node/k8s-lab-w02 --timeout=180s
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl -n kube-flannel get daemonset kube-flannel-ds
kubectl -n kube-system get deployment coredns
```

확인

- 노드 3대 모두 `Ready`, `INTERNAL-IP`가 계획한 IP와 일치
- 워커의 `ROLES`가 `<none>`이어도 정상, 역할 표시용 라벨과 실제 가입 여부는 별개
- Flannel DaemonSet의 desired·ready 모두 3
- CoreDNS desired·ready 일치, 시스템 Pod의 반복 재시작 없음

컨트롤 플레인의 기본 `NoSchedule` taint 유지, 일반 앱은 워커에 배치

## 문제가 생겼을 때

| 증상 | 먼저 확인할 내용 |
|---|---|
| init의 CRI 오류 | `systemctl status containerd`, CRI 활성화, 소켓 경로 |
| init의 cgroup 오류 | containerd `SystemdCgroup = true`와 kubelet 설정 일치 여부 |
| init·join timeout | 노드 IP·경로, API TCP 6443, 이미지 다운로드 |
| join 인증 오류 | 토큰 만료, CA 해시, 노드 시각 |
| CNI 설치 후에도 NotReady | Flannel 로그, 모듈·sysctl, Pod CIDR, 방화벽 |
| kubectl localhost:8080 오류 | ubuntu 계정의 `.kube/config` 존재·권한 |

**문제 노드에서**

```sh
sudo journalctl -u kubelet -n 100 --no-pager
sudo journalctl -u containerd -n 100 --no-pager
```

**컨트롤 플레인에서**

```sh
kubectl describe node k8s-lab-w01
kubectl -n kube-flannel get pods -o wide
kubectl -n kube-flannel logs -l app=flannel -c kube-flannel --tail=100
kubectl get events -A --sort-by=.metadata.creationTimestamp
```

CNI를 바꿔가며 같은 클러스터에 중복 설치하지 않음, 재시도는 [정리 절차](05-cleanup.md)에서 깨끗한 VM 재생성 방식 사용

다음: [앱 배포와 통신 확인](04-deploy-and-check.md)

참고: [kubeadm 클러스터 생성](https://v1-35.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/), [워커 추가](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/adding-linux-nodes/)
