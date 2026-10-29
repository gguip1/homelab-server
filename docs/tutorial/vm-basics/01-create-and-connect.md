# 1. VM 만들고 접속하기

템플릿으로 VM을 만들고 SSH로 접속

## 만들기

```sh
qm clone 9001 900 --name practice
qm set 900 --ipconfig0 ip=10.10.10.90/24,gw=10.10.10.1
qm start 900
qm list
```

확인: `900 practice running`

복제한 VM은 템플릿과 똑같은 상태

VM마다 달라야 하는 IP만 지정

## IP 확인

1~2분 대기 후

```sh
qm guest cmd 900 ping
qm guest cmd 900 network-get-interfaces | grep '"ip-address"'
```

확인: `ping`은 출력 없음, IP 목록에 `10.10.10.90`

## 접속

Mac에서

```sh
ssh -J pve ubuntu@10.10.10.90
```

처음 접속이라 `yes` 입력

확인: 비밀번호 없이 `ubuntu@practice:~$`

`-J pve`는 서버를 거쳐 내부망 VM으로 접속하는 옵션

`exit`로 종료

## 끝

[실습 VM 지우기](README.md#실습-vm-지우기)
