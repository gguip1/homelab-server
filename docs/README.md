# 문서

| 폴더 | 내용 |
|---|---|
| [`setup/`](setup/) | 처음 한 번 하는 설정, 순서대로 진행 |
| [`reference/`](reference/) | 찾아보는 용도: 서버 정보, 명령어 |
| [`tutorial/`](tutorial/) | 따라 하며 익히는 실습 |

## 설정 순서

1. [SSH 키 로그인 설정](setup/01-ssh-key-login.md)
2. [SSH 비밀번호 로그인 끄기](setup/02-ssh-disable-password.md)
3. [Proxmox 저장소 추가와 업데이트](setup/03-proxmox-update.md)
4. [Ubuntu 템플릿 만들기 (게스트 에이전트 포함)](setup/04-ubuntu-template-agent.md)

## 참고

- [서버 정보](reference/server.md)
- [qm 명령어](reference/qm-commands.md)
- [Kubernetes 명령어](reference/k8s-commands.md)

## 튜토리얼

- [VM 기초](tutorial/vm-basics/README.md)
- [Kubernetes 기초 클러스터](tutorial/k8s-basics/README.md), kubeadm 기반 VM 3대 실습, 실행 검증 전
