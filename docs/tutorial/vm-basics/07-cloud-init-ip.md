# 7. cloud-init으로 IP 바꾸기

만든 VM의 IP를 cloud-init 설정으로 변경

시작: [실습 VM 만들기](README.md#실습-vm-만들기)

## 바꾸기

```sh
qm shutdown 900
qm set 900 --ipconfig0 ip=10.10.10.91/24,gw=10.10.10.1
qm cloudinit pending 900
```

확인: `ipconfig0` 항목에 새 값 `10.10.10.91`

`pending`은 아직 VM에 적용되지 않은 변경

## 적용

```sh
qm start 900
```

1~2분 뒤

```sh
qm guest cmd 900 network-get-interfaces | grep '"ip-address"'
```

확인: `10.10.10.91`

## 접속

Mac에서

```sh
ssh -J pve ubuntu@10.10.10.91 hostname
```

확인: `practice`

cloud-init 설정이 바뀌면 VM이 새로 설정된 것으로 처리되어 호스트 키가 바뀔 수 있음

`REMOTE HOST IDENTIFICATION HAS CHANGED`가 나오면 `ssh-keygen -R 10.10.10.91` 후 다시 접속

## 끝

[실습 VM 지우기](README.md#실습-vm-지우기)

`ssh-keygen -R 10.10.10.91`도 함께 실행
