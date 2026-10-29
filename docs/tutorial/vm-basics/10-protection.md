# 10. 삭제 보호

중요한 VM을 실수로 지우지 않게 막기

시작: [실습 VM 만들기](README.md#실습-vm-만들기)

## 보호 켜기

```sh
qm set 900 --protection 1
qm stop 900 && qm destroy 900
```

확인: `protection` 관련 오류로 삭제가 막힘

## 보호 끄고 삭제

```sh
qm set 900 --protection 0
qm destroy 900
qm list
```

확인: 목록에 900 없음

계속 쓸 VM(예: 쿠버네티스 노드)은 만들 때 보호를 켜 두기

## 끝

Mac에서 `ssh-keygen -R 10.10.10.90`
