# 3. CPU·메모리·디스크 늘리기

사양을 바꾸고 VM 안에서 확인

시작: [실습 VM 만들기](README.md#실습-vm-만들기)

## 현재 사양

Mac에서

```sh
ssh -J pve ubuntu@10.10.10.90 'nproc; free -h | grep Mem; df -h /'
```

확인: CPU `2`, 메모리 약 `1.9Gi`, 디스크 약 `30G`

## 바꾸기

```sh
qm shutdown 900
qm set 900 --cores 4 --memory 4096
qm disk resize 900 scsi0 40G
qm start 900
```

## 다시 확인

1분 뒤 Mac에서

```sh
ssh -J pve ubuntu@10.10.10.90 'nproc; free -h | grep Mem; df -h /'
```

확인: CPU `4`, 메모리 약 `3.8Gi`, 디스크 약 `39G`

- CPU, 메모리: 다음 부팅부터 적용
- 디스크: 늘리기만 가능, 늘린 공간은 부팅 때 cloud-init이 자동으로 파티션에 추가

## 끝

[실습 VM 지우기](README.md#실습-vm-지우기)
