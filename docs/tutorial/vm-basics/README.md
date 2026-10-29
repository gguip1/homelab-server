# VM 기초 튜토리얼

VM 명령을 손에 익히기 위한 실습

각 튜토리얼은 실습 VM을 만들고, 실습하고, 지우는 순서로 진행

명령 목록만 필요하면 [qm 명령어](../../reference/qm-commands.md)

## 목록

1. [VM 만들고 접속하기](01-create-and-connect.md)
2. [전원 다루기](02-power.md)
3. [CPU·메모리·디스크 늘리기](03-resize.md)
4. [디스크 추가하기](04-add-disk.md)
5. [스냅샷으로 되돌리기](05-snapshot.md)
6. [연결 복제와 전체 복제](06-clone-types.md)
7. [cloud-init으로 IP 바꾸기](07-cloud-init-ip.md)
8. [게스트 에이전트로 VM 들여다보기](08-guest-agent.md)
9. [콘솔로 접속하기](09-console.md)
10. [삭제 보호](10-protection.md)

## 공통 값

| 항목 | 값 |
|---|---|
| 템플릿 | `9001` ([04](../../setup/04-ubuntu-template-agent.md)) |
| 실습 VMID | `900` |
| 실습 IP | `10.10.10.90` |

서버 명령은 `ssh pve` 접속 후, Mac 명령은 Mac 터미널에서 실행

## 실습 VM 만들기

```sh
qm clone 9001 900 --name practice
qm set 900 --ipconfig0 ip=10.10.10.90/24,gw=10.10.10.1
qm start 900
```

첫 부팅은 1~2분 소요 (cloud-init)

`qm guest cmd 900 ping`이 출력 없이 끝나면 준비 완료

## 실습 VM 지우기

```sh
qm stop 900 && qm destroy 900
```

Mac에서 호스트 키 기록도 삭제 (같은 IP로 다시 만들면 경고가 나오기 때문)

```sh
ssh-keygen -R 10.10.10.90
```
