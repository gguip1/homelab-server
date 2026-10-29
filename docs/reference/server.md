# 서버 정보

2026-10-29 기준

## 접속

| 항목 | 값 |
|---|---|
| 호스트 이름 | `pve` (인증서 이름 `pve.home`) |
| Tailscale 주소 | `100.96.208.74` / `pve.tailb336d1.ts.net` |
| 집 네트워크 주소 | `192.168.0.10` (Wi-Fi) |
| 웹 UI | `https://pve.tailb336d1.ts.net:8006` (자체 서명 인증서, 2028-10-02 만료) |
| SSH | `ssh pve`, 키 로그인만 허용 ([01](../setup/01-ssh-key-login.md), [02](../setup/02-ssh-disable-password.md)) |

## 버전

| 항목 | 값 |
|---|---|
| Proxmox VE | 9.2.24 (`pve-no-subscription` 저장소) |
| 커널 | `7.0.14-26-pve` |
| OS | Debian 13 (trixie) |

## VM과 템플릿

| VMID | 이름 | 비고 |
|---|---|---|
| 100 | `k8s-cp01` | 9000 기반, `10.10.10.10`, 에이전트 없음 |
| 9000 | `ubuntu-2404-base` | 템플릿, 에이전트 없음 |
| 9001 | `ubuntu-2404-agent` | 템플릿, 에이전트·Mac 키 포함 ([04](../setup/04-ubuntu-template-agent.md)) |

VM 내부망: `vmbr0` `10.10.10.0/24`, 게이트웨이 `10.10.10.1`(서버), DHCP 없음

## SSH 설정

| 항목 | 값 |
|---|---|
| 설정 파일 | `/etc/ssh/sshd_config.d/10-key-only.conf` |
| 비밀번호 로그인 | 꺼짐 |
| root 로그인 | 키로만 |
| 키 목록 파일 | `/root/.ssh/authorized_keys` → `/etc/pve/priv/authorized_keys` (심볼릭 링크) |

## 등록된 SSH 키

확인: `ssh pve 'ssh-keygen -lf /root/.ssh/authorized_keys'`

| 키 | 지문 | 비고 |
|---|---|---|
| `homelab-macbook` (ED25519) | `SHA256:upf69y7RnSBksaDD2+FErr+LR86meNV8Tyme40nMRd0` | MacBook |
| `root@pve` (RSA 4096) | `SHA256:aI8YXF+CBDjqRaH3dnwyMA3dNoB+ojOktMhPtPMc5WU` | Proxmox 노드 자체 키, **삭제 금지** |

## Tailscale

| 항목 | 값 |
|---|---|
| tailnet | `tailb336d1.ts.net` |
| 키 만료 | **2027-04-07** (만료를 끄지 않으면 이날 원격 접속 끊김) |
| exit node / 서브넷 라우팅 / Tailscale SSH | 사용 안 함 |

## 열린 포트 (Tailscale 쪽)

| 포트 | 용도 |
|---|---|
| 22 | SSH |
| 111 | rpcbind (Proxmox 기본) |
| 3128 | SPICE 프록시 (Proxmox 기본) |
| 8006 | 웹 UI |
