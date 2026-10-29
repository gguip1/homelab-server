# 5. 스냅샷으로 되돌리기

지금 상태를 저장하고, 망가뜨린 뒤, 되돌리기

시작: [실습 VM 만들기](README.md#실습-vm-만들기)

## 파일 만들기

Mac에서

```sh
ssh -J pve ubuntu@10.10.10.90 'echo hello > ~/note.txt; cat ~/note.txt'
```

확인: `hello`

## 스냅샷 저장

```sh
qm snapshot 900 before-delete
qm listsnapshot 900
```

확인: 목록에 `before-delete`

## 망가뜨리기

Mac에서

```sh
ssh -J pve ubuntu@10.10.10.90 'rm ~/note.txt; ls ~'
```

확인: `note.txt` 없음

## 되돌리기

```sh
qm rollback 900 before-delete --start 1
```

1분 뒤 Mac에서

```sh
ssh -J pve ubuntu@10.10.10.90 'cat ~/note.txt'
```

확인: `hello`

## 스냅샷 삭제

```sh
qm delsnapshot 900 before-delete
qm listsnapshot 900
```

- 업데이트나 설정 변경 전에 찍어 두는 되돌리기 지점
- 백업이 아님, VM을 지우면 스냅샷도 함께 삭제
- 오래 두면 디스크 공간 차지, 쓸 일이 끝나면 삭제

## 끝

[실습 VM 지우기](README.md#실습-vm-지우기)
