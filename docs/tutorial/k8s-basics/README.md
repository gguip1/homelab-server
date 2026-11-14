# Kubernetes 기초 클러스터 실습

Proxmox VM 3대로 Kubernetes 클러스터를 만들고, 앱 배포와 노드 사이 통신까지 확인하는 실습

상태: 설치 명령 작성 및 정적 검토 완료, 실제 홈랩에서 실행 검증 전

## 목표

- 템플릿 복제로 노드 준비
- 컨트롤 플레인 초기화와 워커 가입 과정 이해
- 노드 3대의 `Ready`, CoreDNS와 CNI 동작 확인
- 서로 다른 워커의 Pod 통신, Service DNS, Mac에서 앱 응답 확인

## 목록

1. [VM 준비](01-prepare-vms.md)
2. [노드 공통 설정](02-prepare-nodes.md)
3. [클러스터 생성](03-create-cluster.md)
4. [앱 배포와 통신 확인](04-deploy-and-check.md)
5. [종료와 정리](05-cleanup.md)

앞 장의 상태를 이어서 사용하는 순차 실습, VM 기초 튜토리얼처럼 장마다 VM을 새로 만들지 않음

명령 목록만 필요하면 [Kubernetes 명령어](../../reference/k8s-commands.md)

## 이번 실습의 선택

| 항목 | 실습 구성 | 선택 이유 |
|---|---|---|
| 노드 OS | Ubuntu 24.04, 템플릿 9001 | 기존 준비 환경 재사용 |
| 설치 도구 | kubeadm | 초기화와 가입 과정을 직접 확인 |
| Kubernetes | v1.35 계열, 설치 시 패치 버전 통일 | containerd 1.x 환경을 포함하는 실습 기준, 실행 전 지원 기간 재확인 필요 |
| 컨테이너 런타임 | Ubuntu 저장소의 containerd | kubelet 요청에 따라 컨테이너 실행 |
| CNI | Flannel v0.28.9, VXLAN | 기본 Pod 네트워크 연결 실습 |
| 토폴로지 | 컨트롤 플레인 1대 + 워커 2대 | 역할 분리와 서로 다른 노드의 통신 확인 |
| 클론 | 전체 복제 | 이번 실습 VM의 디스크를 독립적으로 관리 |
| 접속 | Mac → SSH 점프 호스트 pve → VM | 기존 접속 방식 재사용 |
| kubectl 실행 | 컨트롤 플레인의 ubuntu 계정 | Mac에 관리자 kubeconfig 복사 없이 실습 |

kubeadm은 배포판이 아니라 클러스터 부트스트랩 도구, K3s·RKE2 같은 배포판과 선택 관점이 다름

Flannel은 이번 실습용 선택, NetworkPolicy 적용 실습은 별도 구성 필요

버전 고정은 최신 버전 보장이 아님, 실행 시점에 [Kubernetes 지원 버전](https://kubernetes.io/releases/)과 [Flannel 릴리스](https://github.com/flannel-io/flannel/releases) 확인 필요

## VM과 네트워크 계획

아래 값은 생성 계획이며 현재 서버 상태가 아님, 사용 중인 VMID·IP이면 실습 전체에서 일관되게 다른 값 사용

| VMID | 노드 이름 | IP | CPU | 메모리 | 디스크 |
|---|---|---|---|---|---|
| 910 | k8s-lab-cp01 | 10.10.10.110 | 2코어 | 4GiB | 템플릿 크기 상속 |
| 911 | k8s-lab-w01 | 10.10.10.111 | 2코어 | 2GiB | 템플릿 크기 상속 |
| 912 | k8s-lab-w02 | 10.10.10.112 | 2코어 | 2GiB | 템플릿 크기 상속 |

- VM 네트워크: `10.10.10.0/24`, 게이트웨이 `10.10.10.1`
- Pod 네트워크: `10.244.0.0/16`
- Service 네트워크: `10.96.0.0/12`
- VM·Pod·Service·LAN·VPN 경로가 서로 겹치지 않는지 사전 확인
- 메모리 합계 8GiB 외에 Proxmox와 기존 VM용 여유 필요, 전체 복제용 저장 공간 확인 필요
- 기존 VM 100과 실습 VM 900·901 등은 변경·삭제 대상에서 제외

## 어디에서 실행하는지 확인

| 위치 | 프롬프트 예시 | 역할 |
|---|---|---|
| Mac | `gguip@...` | SSH 접속, 브라우저·curl 확인 |
| Proxmox | `root@pve:~#` | VM 생성·종료·삭제 |
| 컨트롤 플레인 | `ubuntu@k8s-lab-cp01:~$` | kubeadm init, kubectl |
| 워커 | `ubuntu@k8s-lab-w01:~$` 등 | 노드 설정, kubeadm join |

`ssh -J pve ...`는 Mac에서 실행, VM에 들어간 뒤에는 해당 VM 명령만 실행

## 나중에 정할 운영 구성

| 결정 항목 | 검토할 내용 |
|---|---|
| 설치 방식·배포판 | kubeadm 유지, K3s·RKE2 등 비교 |
| 런타임·버전 정책 | containerd 또는 CRI-O, 업그레이드·지원 기간 |
| CNI | Flannel·Calico·Cilium, NetworkPolicy와 네트워크 기능 |
| 노드·HA | 노드 사양과 수, 컨트롤 플레인 다중화, API 엔드포인트 |
| 주소 계획 | 노드·Pod·Service CIDR, DNS, LAN·VPN 경로 충돌 |
| 앱 외부 접속 | Gateway API·Ingress, LoadBalancer 구현, DNS·TLS |
| 영구 저장소 | CSI·StorageClass, 로컬·공유 스토리지, 데이터 복구 |
| 운영·자동화 | IaC, GitOps, 비밀정보 관리, 모니터링, 백업·복원 |

CNI 하나와 주소 범위는 이번 생성 전에 선택 필요, 운영용 최종 선택은 이후 실습에서 재검토

단일 컨트롤 플레인과 단일 물리 Proxmox 서버 구성으로 HA 보장 없음, 이 실습 클러스터를 운영 서버로 간주하지 않음

## 공식 참고

- [kubeadm 개요](https://kubernetes.io/docs/reference/setup-tools/kubeadm/)
- [Kubernetes 1.35 설치 준비](https://v1-35.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
- [Kubernetes 1.35 클러스터 생성](https://v1-35.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
- [Flannel](https://github.com/flannel-io/flannel)
