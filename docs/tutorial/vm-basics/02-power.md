# 2. 전원 다루기

끄기, 켜기, 재부팅, 일시정지의 차이

시작: [실습 VM 만들기](README.md#실습-vm-만들기)

## 정상 종료

```sh
qm shutdown 900
qm status 900
```

확인: 몇 초 뒤 `status: stopped`

OS가 스스로 정리하고 꺼짐

평소에는 이 방법 사용

## 강제 종료

```sh
qm start 900
qm stop 900
qm status 900
```

확인: 바로 `status: stopped`

전원을 뽑은 것과 같음

VM이 응답하지 않을 때만 사용

## 재부팅

```sh
qm start 900
qm reboot 900
qm status 900
```

확인: `status: running`

`qm reset`은 강제 재부팅 (리셋 버튼)

## 일시정지

```sh
qm suspend 900
qm status 900 --verbose | grep qmpstatus
```

확인: `qmpstatus: paused`

```sh
qm resume 900
qm status 900 --verbose | grep qmpstatus
```

확인: `qmpstatus: running`

메모리 상태를 그대로 둔 채 멈춤

다시 실행하면 멈춘 지점부터 이어서 진행

## 끝

[실습 VM 지우기](README.md#실습-vm-지우기)
