# 6. 연결 복제와 전체 복제

두 복제 방식의 속도와 디스크 차이

이 튜토리얼은 실습 VM을 직접 2개 생성 (900, 901, 켜지 않음)

## 연결 복제

```sh
time qm clone 9001 900 --name practice-linked
```

## 전체 복제

```sh
time qm clone 9001 901 --name practice-full --full
```

확인: 전체 복제가 더 오래 걸림 (`real` 시간 비교)

## 디스크 비교

```sh
lvs -o lv_name,lv_size,origin,data_percent pve | grep -E "base-9001|vm-90[01]-disk-1"
```

확인:

- `vm-900-disk-1`의 `Origin`이 `base-9001-disk-1` → 템플릿 디스크를 공유
- `vm-901-disk-1`은 `Origin` 없음 → 독립 디스크

| | 연결 복제 | 전체 복제 (`--full`) |
|---|---|---|
| 속도, 공간 | 빠르고 적게 사용 | 디스크 전체 복사 |
| 템플릿 | 이 VM이 있는 동안 템플릿 삭제 불가 | 템플릿과 무관 |
| 용도 | 잠깐 쓰는 테스트 VM | 계속 쓰는 VM |

## 끝

```sh
qm destroy 900
qm destroy 901
```

켜지 않았으므로 `stop` 불필요
