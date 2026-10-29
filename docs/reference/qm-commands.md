# qm 명령어

Proxmox VM 관리 명령, 서버(`ssh pve`)에서 실행, `<id>`는 VMID

## 보기

```sh
qm list                         # VM 목록
qm status <id>                  # 켜짐/꺼짐
qm config <id>                  # 설정
qm pending <id>                 # 다음 부팅에 적용될 변경
```

## 켜기 / 끄기

```sh
qm start <id>                   # 켜기
qm shutdown <id>                # 정상 종료 (평소에 사용)
qm stop <id>                    # 강제 종료 (전원 뽑기)
qm reboot <id>                  # 정상 재부팅
qm reset <id>                   # 강제 재부팅
qm suspend <id>                 # 일시정지
qm resume <id>                  # 일시정지 해제
```

## 만들기 / 지우기

```sh
qm clone <템플릿id> <새id> --name <이름>          # 연결 복제 (빠름, 템플릿에 의존)
qm clone <템플릿id> <새id> --name <이름> --full   # 전체 복제 (독립 디스크)
qm template <id>                                # 템플릿으로 전환 (되돌릴 수 없음)
qm stop <id> && qm destroy <id>                 # 삭제 (꺼진 상태 필요, 되돌릴 수 없음)
qm set <id> --protection 1                      # 삭제 방지
```

## 설정 바꾸기

```sh
qm set <id> --cores 4 --memory 8192             # CPU, 메모리 (MB)
qm set <id> --onboot 1                          # 서버 부팅 때 자동 시작
qm set <id> --name <이름>                        # 이름 변경
qm disk resize <id> scsi0 +10G                  # 디스크 늘리기 (줄이기 불가)
qm set <id> --scsi1 local-lvm:10                # 10GB 디스크 추가
```

## cloud-init

```sh
qm set <id> --ipconfig0 ip=10.10.10.<N>/24,gw=10.10.10.1   # IP
qm set <id> --ciuser ubuntu                                # 계정
qm set <id> --sshkeys <공개키 파일>                          # SSH 공개키
qm set <id> --nameserver 1.1.1.1                           # DNS
qm cloudinit dump <id> user                                # VM이 받을 설정 확인
qm cloudinit pending <id>                                  # 아직 적용 안 된 변경
```

## 게스트 에이전트

VM 안에 `qemu-guest-agent` 필요

```sh
qm guest cmd <id> ping                          # 응답 확인 (출력 없으면 정상)
qm guest cmd <id> network-get-interfaces        # IP
qm guest cmd <id> get-osinfo                    # OS 정보
qm guest exec <id> -- <명령>                     # VM 안에서 명령 실행
qm guest passwd <id> <사용자>                     # 비밀번호 변경
```

`qm agent <id> ...`는 `qm guest cmd <id> ...`의 옛 이름

## 스냅샷

```sh
qm snapshot <id> <이름>                          # 저장
qm listsnapshot <id>                            # 목록
qm rollback <id> <이름>                          # 되돌리기
qm delsnapshot <id> <이름>                       # 삭제
```

## 콘솔

```sh
qm terminal <id>                                # 시리얼 콘솔 (나가기: Ctrl+O)
```

## 문제가 생겼을 때

```sh
qm unlock <id>                                  # 작업 중단 후 잠긴 VM 풀기
qm help <명령> --verbose                         # 명령 상세 설명
```
