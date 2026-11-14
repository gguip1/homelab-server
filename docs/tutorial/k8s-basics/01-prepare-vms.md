# 1. VM 준비

시작: [구성과 주소 계획](README.md) 확인, [Ubuntu 템플릿](../../setup/04-ubuntu-template-agent.md) 준비 완료

## 1. 사용 중인 자원 확인

**Mac에서**

```sh
ssh pve
```

**Proxmox에서**

```sh
qm list
pvesm status
free -h
```

확인: VMID 910·911·912 미사용, 템플릿 9001 존재, 저장 공간과 메모리 여유

IP `10.10.10.110`~`10.10.10.112`는 VM 설정·주소 관리 기록과 켜진 장치에서 미사용 확인 필요, ping 무응답만으로 미사용 판단 불가

기존 VM 100은 상태 확인 없이 재사용하지 않음

## 2. 템플릿 전체 복제

**Proxmox에서**, 각 명령 성공 확인 후 다음 명령 진행

```sh
qm clone 9001 910 --name k8s-lab-cp01 --full
qm clone 9001 911 --name k8s-lab-w01 --full
qm clone 9001 912 --name k8s-lab-w02 --full
```

`already exists`면 기존 VM 확인 후 다른 번호 선택, 덮어쓰기나 자동 삭제 금지

## 3. 사양과 IP 지정

**Proxmox에서**

```sh
qm set 910 --cores 2 --memory 4096 --ipconfig0 ip=10.10.10.110/24,gw=10.10.10.1 --protection 1
qm set 911 --cores 2 --memory 2048 --ipconfig0 ip=10.10.10.111/24,gw=10.10.10.1 --protection 1
qm set 912 --cores 2 --memory 2048 --ipconfig0 ip=10.10.10.112/24,gw=10.10.10.1 --protection 1
qm start 910
qm start 911
qm start 912
```

첫 부팅과 cloud-init 완료까지 대기

## 4. Mac에서 접속 확인

Proxmox 터미널에서 `exit`로 Mac 복귀 후 실행

```sh
ssh -J pve ubuntu@10.10.10.110 'cloud-init status --wait; hostname; ip -br addr'
ssh -J pve ubuntu@10.10.10.111 'cloud-init status --wait; hostname; ip -br addr'
ssh -J pve ubuntu@10.10.10.112 'cloud-init status --wait; hostname; ip -br addr'
```

확인: `status: done`, 각 노드 이름과 지정 IP, 노드별 cloud-init 오류 없음

호스트 이름이 계획과 다르면 클러스터 생성 전에 해당 VM에서 `sudo hostnamectl set-hostname <계획한-노드-이름>` 실행 후 `/etc/hosts`의 이전 호스트 이름도 수정

호스트 키 변경 경고 시 VM 재생성 여부와 콘솔의 호스트 키 지문 확인 후 해당 IP의 `known_hosts` 항목만 갱신

## 5. 노드 고유값과 통신 확인

**Mac에서**, 노드별 출력 비교

```sh
for ip in 10.10.10.110 10.10.10.111 10.10.10.112; do
  ssh -J pve ubuntu@"$ip" 'hostname; cat /etc/machine-id; sudo cat /sys/class/dmi/id/product_uuid; ip -br link'
done
```

확인: 호스트 이름·machine-id·product_uuid·NIC MAC 주소가 노드별로 서로 다름, 같으면 템플릿·복제 설정 수정 후 재생성

각 VM에 접속하여 나머지 두 노드로 `ping -c 2 <상대-IP>` 확인, DNS와 인터넷도 확인

```sh
getent hosts registry.k8s.io
curl -I https://registry.k8s.io/v2/
```

`curl`이 없으면 다음 장의 패키지 설치 후 재확인, 응답 코드보다 DNS 해석·TLS 연결 가능 여부 확인

VM outbound는 Proxmox의 기존 NAT·라우팅에 의존, 다운로드 실패 시 클러스터 설치 전에 해결 필요

## 방화벽 확인

Proxmox Datacenter·노드·VM 방화벽과 Ubuntu의 `sudo ufw status` 확인, 필요한 통신만 허용

| 통신 | 필요한 범위 |
|---|---|
| SSH TCP 22 | 기존 관리 접속 경로에서 각 VM |
| Kubernetes API TCP 6443 | 워커에서 컨트롤 플레인 |
| kubelet TCP 10250 | 컨트롤 플레인에서 각 노드 |
| Flannel VXLAN UDP 8472 | 실습 노드 세 대 사이 양방향 |
| Pod 트래픽 전달 | 노드의 CNI 브리지·VXLAN과 Pod CIDR 관련 forwarding |
| DNS·패키지·이미지 다운로드 | 기존 DNS 및 인터넷으로 나가는 통신 |

ping 성공만으로 위 통신 허용을 보장하지 않음, 방화벽이 기본 차단이면 설치 전 해당 규칙 구성 필요

etcd TCP 2379~2380은 컨트롤 플레인 내부 용도, 이번 실습에서 LAN·인터넷 공개 불필요

NodePort·외부 LoadBalancer는 이번 실습에서 사용하지 않음, 공유기 포트포워딩 불필요

참고: [Kubernetes 포트](https://kubernetes.io/docs/reference/networking/ports-and-protocols/), [Flannel VXLAN](https://github.com/flannel-io/flannel/blob/v0.28.9/Documentation/backends.md#vxlan)

다음: [노드 공통 설정](02-prepare-nodes.md)
