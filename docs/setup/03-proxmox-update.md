# Proxmox 저장소 추가와 업데이트

Proxmox 무료 저장소를 추가하고 서버 전체를 업데이트

완료 후 `apt full-upgrade` 한 번으로 Debian과 Proxmox 함께 업데이트

## 배경

Proxmox VE는 Debian에 Proxmox 커널과 패키지를 더한 구성, 업데이트도 두 곳에서 받음

| 저장소 | 받는 것 |
|---|---|
| Debian | bash, ssh, systemd 같은 기본 OS 패키지 |
| Proxmox | 커널, VM 실행(QEMU), 웹 UI, `qm` 등 |

설치 직후에는 유료 저장소(`pve-enterprise`)만 등록된 상태, 구독 없이 쓰려면 유료 저장소를 끄고 무료 저장소(`pve-no-subscription`) 추가 필요

## 1. 저장소 상태 확인

```sh
ssh pve
grep -H -E "^(URIs|Components|Enabled)" /etc/apt/sources.list.d/*.sources
```

- `pve-enterprise`, `ceph` 항목에 `Enabled: false`가 있어야 함, 없으면 웹 UI의 `pve` → Updates → Repositories에서 **Disable**
- `pve-no-subscription`이 이미 있으면 4단계로 이동

## 2. 무료 저장소 추가

아래 7줄을 한 번에 붙여 넣기

```sh
cat > /etc/apt/sources.list.d/proxmox.sources <<'EOF'
Types: deb
URIs: http://download.proxmox.com/debian/pve
Suites: trixie
Components: pve-no-subscription
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
EOF
```

`trixie`는 Debian 13(Proxmox 9)의 이름, 버전이 다르면 `cat /etc/os-release`의 `VERSION_CODENAME` 값 사용

## 3. 연결 확인

```sh
apt update
apt policy pve-manager
```

`apt update`가 `Err:` 없이 끝나고, `apt policy`에 `download.proxmox.com ... pve-no-subscription`이 보이면 완료

## 4. tmux 열기

업데이트 중 SSH가 끊기면 업데이트도 중단됨, tmux 안에서 실행하면 끊겨도 계속 진행

```sh
apt install tmux
tmux new -s upgrade
```

| 상황 | 방법 |
|---|---|
| 접속이 끊김 | `ssh pve` 후 `tmux attach -t upgrade` |
| 실행 중에 빠져나오기 | `Ctrl+b` → `d` |
| 단축키 목록 | `Ctrl+b` → `?` |

## 5. 업데이트

```sh
apt full-upgrade
```

- `[Y/n]`에 `Y`
- 변경 안내(`apt-listchanges: News`)가 뜨면 읽고 `q`
- 설정 파일을 바꿀지 물으면 기본값(현재 파일 유지, `N`)
- `apt upgrade`가 아닌 `apt full-upgrade` 사용, 패키지 추가·제거가 필요한 업데이트까지 처리

끝나면 `exit`로 tmux 종료

## 6. 재부팅

커널이 바뀌었으면 재부팅 필요

```sh
dpkg -l "proxmox-kernel-*-pve-signed" | awk '/^ii/{print $2}'
uname -r
```

목록에 현재 커널(`uname -r`)보다 새 버전이 있으면 `reboot`, 1~2분 뒤 다시 `ssh pve`

## 7. 확인

```sh
uname -r
pveversion
apt list --upgradable
systemctl --failed
qm list
```

- `uname -r`이 새 커널 버전
- 남은 업데이트와 실패한 서비스 없음
- 꺼진 VM은 `qm start <VMID>`로 켜기 (`onboot`가 꺼진 VM은 재부팅 후 꺼진 상태)

## 이후 정기 업데이트

저장소 추가는 한 번만, 이후에는 4~7단계 반복

```sh
ssh pve
tmux new -s upgrade
apt update && apt full-upgrade
```

## 문제 해결

| 증상 | 확인할 것 |
|---|---|
| `apt update`에서 `401 Unauthorized` | 유료 저장소가 켜져 있음, 1단계에서 끄기 |
| `Could not get lock /var/lib/dpkg/lock-frontend` | 다른 apt가 실행 중, `tmux ls`로 남은 세션 확인 |
| 업데이트가 중간에 끊김 | `dpkg --audit` 확인, 출력이 있으면 `dpkg --configure -a` 후 `apt full-upgrade` 다시 실행 |
