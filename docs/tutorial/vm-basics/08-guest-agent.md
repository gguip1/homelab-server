# 8. 게스트 에이전트로 VM 들여다보기

SSH 없이 서버에서 VM 안의 정보를 보고 명령 실행

시작: [실습 VM 만들기](README.md#실습-vm-만들기)

## 정보 보기

```sh
qm guest cmd 900 get-osinfo
qm guest cmd 900 get-host-name
qm guest cmd 900 get-fsinfo
```

확인: Ubuntu 버전, 호스트 이름 `practice`, 디스크 사용량

## 명령 실행

```sh
qm guest exec 900 -- df -h /
qm guest exec 900 -- uptime
```

확인: 결과가 `"out-data"` 안에 출력

`--` 뒤가 VM 안에서 실행할 명령

## 정상 종료 요청

```sh
qm guest cmd 900 shutdown
qm status 900
```

확인: 몇 초 뒤 `status: stopped`

`qm shutdown`도 에이전트가 있으면 이 방식 사용

## 끝

[실습 VM 지우기](README.md#실습-vm-지우기)
