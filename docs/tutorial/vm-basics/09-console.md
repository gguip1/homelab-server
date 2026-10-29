# 9. 콘솔로 접속하기

SSH가 안 될 때 시리얼 콘솔로 VM에 들어가기

시작: [실습 VM 만들기](README.md#실습-vm-만들기)

## 비밀번호 정하기

`ubuntu` 계정은 SSH 키로만 로그인하도록 만들어져 비밀번호가 없음

콘솔 로그인에는 비밀번호가 필요하므로 에이전트로 지정

```sh
qm guest passwd 900 ubuntu
```

새 비밀번호를 두 번 입력

## 콘솔 접속

```sh
qm terminal 900
```

1. `Enter`를 한 번 눌러 로그인 화면 표시
2. `practice login:`에 `ubuntu`, 이어서 비밀번호 입력
3. 확인: `ubuntu@practice:~$`
4. `exit`로 로그아웃
5. `Ctrl+O`로 콘솔 종료

네트워크와 상관없이 접속되므로 IP 설정을 잘못했을 때 유용

웹 UI의 VM → Console도 같은 화면

## 끝

[실습 VM 지우기](README.md#실습-vm-지우기)
