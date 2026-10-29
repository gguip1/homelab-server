# SSH 비밀번호 로그인 끄기

서버 SSH를 키 로그인만 허용하도록 변경, 키가 없는 기기는 SSH 접속 불가

Proxmox 웹 UI 로그인(root 비밀번호)은 영향 없음

## 준비

- [01](01-ssh-key-login.md) 완료, `ssh pve`가 비밀번호 없이 접속되는 상태
- 서버 root 비밀번호 (키를 잃었을 때 웹 UI로 복구하는 데 필요)

## 1. 서버 접속 (터미널 A)

```sh
ssh pve
```

**5단계 확인이 끝날 때까지 이 터미널은 닫지 않음**, 문제가 생기면 여기서 되돌림

## 2. 설정 파일 만들기

아래 5줄을 한 번에 붙여 넣기

```sh
cat > /etc/ssh/sshd_config.d/10-key-only.conf <<'EOF'
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin prohibit-password
EOF
```

| 설정 | 의미 |
|---|---|
| `PasswordAuthentication no` | 비밀번호 로그인 끔 |
| `KbdInteractiveAuthentication no` | 비밀번호를 받는 다른 통로(키보드 대화형 인증)도 끔 |
| `PermitRootLogin prohibit-password` | root는 키로만 로그인 |

원본 `/etc/ssh/sshd_config`는 수정하지 않음

## 3. 적용

```sh
sshd -t && systemctl reload ssh
```

아무것도 출력되지 않으면 정상, 문법 오류가 있으면 적용되지 않음

## 4. 적용 값 확인

```sh
sshd -T | grep -E "^(passwordauthentication|kbdinteractiveauthentication|permitrootlogin) "
```

```
permitrootlogin without-password
passwordauthentication no
kbdinteractiveauthentication no
```

`without-password`는 `prohibit-password`와 같은 의미

## 5. 새 터미널에서 확인 (터미널 B)

```sh
ssh pve 'echo key-login-ok'
```

`key-login-ok` 출력 확인

```sh
ssh -o PubkeyAuthentication=no pve
```

비밀번호를 묻지 않고 `Permission denied (publickey).`가 나오면 성공, 터미널 A는 `exit`로 종료

## 되돌리기

```sh
rm /etc/ssh/sshd_config.d/10-key-only.conf && systemctl reload ssh
```

터미널 A가 닫혔으면 웹 UI의 Shell에서 실행

## 키를 잃어버렸을 때

1. 새 기기에서 [01](01-ssh-key-login.md)의 1~3단계로 키 생성
2. 웹 UI(`https://pve.tailb336d1.ts.net:8006`)에 root 비밀번호로 로그인, Tailscale이 안 되면 집 네트워크에서 `https://192.168.0.10:8006`
3. `pve` 노드 → **Shell**에서 새 공개키 추가

   ```sh
   echo '<새 공개키 한 줄>' >> /etc/pve/priv/authorized_keys
   ```

4. 잃어버린 키 줄은 [서버 정보](../reference/server.md)의 지문을 보고 삭제

키가 유출된 경우(Mac 도난 등)에는 Tailscale 관리 콘솔에서 그 기기부터 제거
