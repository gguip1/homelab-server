# SSH 키 로그인 설정

Mac에서 Proxmox 서버(`pve`)에 SSH 키로 접속하는 설정

완료 후 `ssh pve`만으로 접속 가능

## 준비

- Mac의 Tailscale 연결 (메뉴바 아이콘 → Connected)
- 서버 root 비밀번호 (4단계에서 한 번 입력)

| 항목 | 값 |
|---|---|
| 서버 이름 (Tailscale) | `pve.tailb336d1.ts.net` |
| 사용자 | `root` |
| 키 파일 | `~/.ssh/id_ed25519` |

서버는 IP 대신 Tailscale 이름으로 지정, 서버를 다시 설치해 IP가 바뀌어도 기기 이름이 `pve`면 설정 변경 불필요

## 1. 키 만들기

기존 키 확인

```sh
ls ~/.ssh/id_ed25519
```

`No such file or directory`가 나오면 아래 명령으로 생성, 이미 있으면 2단계로 이동 (덮어쓰면 기존 키로 접속하던 곳에 접속 불가)

```sh
ssh-keygen -t ed25519 -C "homelab-macbook" -f ~/.ssh/id_ed25519
```

- passphrase: 키 파일을 잠그는 비밀번호, 입력 권장 (화면에 글자가 안 보이는 것이 정상)
- **영문 입력 상태에서 입력**, 한글 상태면 다른 값이 들어감
- 잊으면 복구 불가, 비밀번호 관리 앱에 기록

`ls ~/.ssh/id_ed25519 ~/.ssh/id_ed25519.pub`로 두 파일이 모두 보이면 완료

- `id_ed25519`: 비밀키 (Mac에만 보관)
- `id_ed25519.pub`: 공개키 (서버에 등록)

## 2. passphrase를 키체인에 저장

passphrase가 없으면 생략

```sh
ssh-add --apple-use-keychain ~/.ssh/id_ed25519
```

`Identity added`가 나오면 완료, 이후 passphrase를 묻지 않음

## 3. 접속 설정 추가

```sh
cat >> ~/.ssh/config <<'EOF'

Host pve
    HostName pve.tailb336d1.ts.net
    User root
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
    AddKeysToAgent yes
    UseKeychain yes
EOF
chmod 600 ~/.ssh/config
```

`AddKeysToAgent`, `UseKeychain`: 재부팅 후에도 키체인에 저장한 passphrase를 자동 사용

## 4. 서버에 공개키 등록

```sh
ssh-copy-id -i ~/.ssh/id_ed25519.pub pve
```

1. 처음 접속이면 `Are you sure you want to continue connecting` → `yes`
2. `password:`에 서버 root 비밀번호 입력
3. `Number of key(s) added: 1`이 나오면 완료

**[02](02-ssh-disable-password.md)로 비밀번호 로그인을 끈 서버에서는 실패**, 이미 등록된 기기에서 `ssh pve`로 들어가거나 웹 UI의 Shell에서 직접 추가

```sh
echo '<공개키 한 줄>' >> /etc/pve/priv/authorized_keys
```

`>>` 두 개 사용, `>` 하나면 다른 키가 모두 지워짐

## 5. 접속 확인

```sh
ssh pve
```

passphrase와 비밀번호 질문 없이 `root@pve:~#`가 나오면 성공, `exit`로 종료

## 문제 해결

| 증상 | 확인할 것 |
|---|---|
| `Connection timed out` | Mac의 Tailscale 연결 (`tailscale status`에 `pve`가 보이는지) |
| `Enter passphrase for key`를 매번 물음 | 2단계 다시 실행 |
| 서버 비밀번호를 물음 | passphrase가 틀렸거나 4단계 실패, 2단계, 4단계 순서로 다시 확인 |
| `Bad passphrase, try again` | 입력기가 한글인지 확인, 그래도 안 되면 복구 불가, 1단계부터 새 키 생성 (덮어쓰기 `y`) |
| `Permission denied (publickey...)` | 3단계 `IdentityFile` 경로가 1단계의 파일과 같은지 |
| `Could not resolve hostname` | Tailscale 관리 콘솔에서 기기 이름이 `pve`인지, MagicDNS가 켜져 있는지 |
| `REMOTE HOST IDENTIFICATION HAS CHANGED` | 서버를 재설치했다면 `ssh-keygen -R pve.tailb336d1.ts.net` 후 다시 접속 |
