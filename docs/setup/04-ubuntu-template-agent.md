# Ubuntu 템플릿 만들기 (게스트 에이전트 포함)

Ubuntu 24.04 클라우드 이미지에 게스트 에이전트를 넣고, Mac SSH 키를 등록한 템플릿(VM 9001) 생성

완료 후 템플릿을 복제하고 IP만 정하면 바로 사용 가능한 VM 생성

## 배경

게스트 에이전트(`qemu-guest-agent`)는 VM 안에서 Proxmox와 통신하는 프로그램

| 기능 | 에이전트 없음 | 에이전트 있음 |
|---|---|---|
| VM IP 확인 | 웹 UI·API로 확인 불가 | 확인 가능 (Terraform 등 자동화에 필요) |
| 종료 | 가상 전원 버튼 신호만 전달 | OS에 정상 종료 요청 |
| 백업 | 쓰는 도중의 상태로 백업될 수 있음 | 파일시스템을 잠깐 멈추고 백업 |

Ubuntu 클라우드 이미지에는 에이전트가 없음, VM 설정의 `agent: 1`만으로는 동작하지 않음

모든 VM에 공통인 프로그램(에이전트)은 이미지에 넣고, VM마다 다른 값(계정, SSH 키, IP)은 cloud-init 설정에 둠

| 템플릿 | 에이전트 | SSH 키 |
|---|---|---|
| 9000 `ubuntu-2404-base` | 없음 | 없음 (비밀번호) |
| **9001 `ubuntu-2404-agent`** | 있음 | Mac 공개키 |

## 1. 클라우드 이미지 받기

이미 받은 파일이 있으면 생략

```sh
ssh pve
cd /var/lib/vz/template
wget https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img
```

## 2. 이미지 수정 도구 설치

```sh
apt install libguestfs-tools
```

## 3. 사본에 에이전트 넣기

원본은 보관하고 사본을 수정

```sh
cp /var/lib/vz/template/noble-server-cloudimg-amd64.img /var/lib/vz/template/noble-agent.img
virt-customize -a /var/lib/vz/template/noble-agent.img --install qemu-guest-agent --truncate /etc/machine-id
```

- `--install qemu-guest-agent`: 이미지 안에 에이전트 설치 (인터넷 필요)
- `--truncate /etc/machine-id`: 기기 고유 ID를 비움, 복제 VM마다 첫 부팅 때 새 ID 생성

마지막에 `Finishing off`가 나오면 완료

```sh
virt-ls -a /var/lib/vz/template/noble-agent.img /usr/sbin/ | grep qemu-ga
```

`qemu-ga`가 나오면 성공

## 4. Mac 공개키를 서버로 복사

**Mac 터미널에서** 실행

```sh
scp ~/.ssh/id_ed25519.pub pve:/root/homelab-macbook.pub
```

## 5. VM 9001 만들기

한 명령이므로 통째로 붙여 넣기

```sh
qm create 9001 --name ubuntu-2404-agent --ostype l26 \
  --machine q35 --bios ovmf \
  --efidisk0 local-lvm:1,efitype=4m,ms-cert=2023k,pre-enrolled-keys=1 \
  --cpu host --cores 2 --memory 2048 \
  --scsihw virtio-scsi-single \
  --scsi0 local-lvm:0,import-from=/var/lib/vz/template/noble-agent.img,iothread=1 \
  --ide0 local-lvm:cloudinit --boot order=scsi0 \
  --net0 virtio,bridge=vmbr0,firewall=1 \
  --serial0 socket --vga serial0 \
  --agent 1
```

| 옵션 | 의미 |
|---|---|
| `--machine q35 --bios ovmf` | 최신 가상 메인보드, UEFI 부팅 |
| `--scsi0 ...import-from=...` | 에이전트를 넣은 이미지를 첫 번째 디스크로 연결 |
| `--ide0 local-lvm:cloudinit` | cloud-init 설정 CD |
| `--net0 ...bridge=vmbr0` | VM 내부망(`10.10.10.0/24`) 연결 |
| `--serial0 socket --vga serial0` | 시리얼 콘솔 (클라우드 이미지 기본 방식) |
| `--agent 1` | 게스트 에이전트 통신 켜기 |

## 6. 디스크 늘리기

```sh
qm disk resize 9001 scsi0 32G
```

복제 VM의 첫 부팅 때 cloud-init이 늘어난 공간을 자동 사용

## 7. cloud-init 설정

```sh
qm set 9001 --ciuser ubuntu --sshkeys /root/homelab-macbook.pub --nameserver 1.1.1.1
qm cloudinit dump 9001 user
```

`ssh_authorized_keys:` 아래에 `homelab-macbook` 키가 보이면 완료, 키는 VM 설정에 저장되므로 파일 삭제

```sh
rm /root/homelab-macbook.pub
```

IP(`ipconfig0`)는 VM마다 달라서 템플릿에는 넣지 않음

## 8. 템플릿으로 전환

```sh
qm template 9001
```

## 9. 테스트

```sh
qm clone 9001 101 --name template-test
qm set 101 --ipconfig0 ip=10.10.10.11/24,gw=10.10.10.1
qm start 101
```

1~2분 뒤 (첫 부팅 때 패키지 업데이트로 시간 소요):

```sh
qm guest cmd 101 ping
qm guest cmd 101 network-get-interfaces | grep '"ip-address"'
```

`ping`은 출력 없음, IP 목록에 `10.10.10.11` 있으면 성공

**Mac 터미널에서:**

```sh
ssh -J pve ubuntu@10.10.10.11 'hostname; systemctl is-active qemu-guest-agent'
```

처음 접속이라 `yes`, 비밀번호 없이 `template-test`, `active`가 나오면 성공

테스트 VM 삭제:

```sh
qm stop 101 && qm destroy 101
```

Mac에서 접속했다면 `ssh-keygen -R 10.10.10.11`로 호스트 키 기록도 삭제

## 문제 해결

| 증상 | 확인할 것 |
|---|---|
| 7단계 후 `qm config 9001`에 `ciuser`, `sshkeys`가 없음 | 공개키 파일이 없어 `qm set` 전체가 실패한 것, 4단계 후 7단계 다시 실행 |
| `QEMU guest agent is not running` | 첫 부팅 중, 1~2분 대기 |
| SSH가 비밀번호를 물음 | `qm cloudinit dump <VMID> user`에 Mac 공개키가 있는지 확인 |
| `REMOTE HOST IDENTIFICATION HAS CHANGED` | 같은 IP로 VM을 다시 만든 경우, `ssh-keygen -R <IP>` |
