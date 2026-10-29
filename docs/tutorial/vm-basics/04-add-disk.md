# 4. 디스크 추가하기

VM에 두 번째 디스크를 붙이고 VM 안에서 사용

시작: [실습 VM 만들기](README.md#실습-vm-만들기)

## 붙이기

VM이 켜진 상태에서

```sh
qm set 900 --scsi1 local-lvm:10
qm config 900 | grep scsi1
```

확인: `scsi1: local-lvm:vm-900-disk-2,size=10G`

`local-lvm:10`은 `local-lvm` 저장소에 10GB 새 디스크 생성

## VM 안에서 확인

Mac에서

```sh
ssh -J pve ubuntu@10.10.10.90 'lsblk'
```

확인: `sdb 10G`가 새로 보임

## 사용하기

```sh
ssh -J pve ubuntu@10.10.10.90 'sudo mkfs.ext4 /dev/sdb && sudo mkdir -p /data && sudo mount /dev/sdb /data && df -h /data'
```

확인: `/dev/sdb ... /data`

- `mkfs.ext4`: 디스크를 ext4 파일시스템으로 포맷
- `mount`: `/data` 폴더에 연결
- 재부팅하면 연결이 풀림 (계속 쓰려면 `/etc/fstab`에 등록 필요)

## 끝

[실습 VM 지우기](README.md#실습-vm-지우기)

VM을 지우면 추가한 디스크도 함께 삭제
