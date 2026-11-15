# 5. 종료와 정리

앱만 지우기, 클러스터 보관, VM 전체 삭제 중 목적에 맞는 방법 선택

## 앱만 정리

**컨트롤 플레인에서**

```sh
kubectl delete namespace k8s-practice
rm -f ~/k8s-practice-web.yaml
```

확인: 실습 앱과 netcheck 제거, Kubernetes 클러스터는 유지

가입 완료 후 더 이상 필요 없는 bootstrap 토큰 확인

```sh
sudo kubeadm token list
```

필요 없는 토큰만 `sudo kubeadm token delete <TOKEN-ID>`로 삭제, 토큰 삭제가 이미 가입한 워커의 연결을 끊지는 않음

## 클러스터를 남겨두고 종료

**Mac에서**

```sh
ssh pve
```

**Proxmox에서**, 워커부터 종료하고 컨트롤 플레인은 마지막에 종료

```sh
qm shutdown 911 --timeout 120
qm shutdown 912 --timeout 120
qm shutdown 910 --timeout 120
qm status 910
qm status 911
qm status 912
```

확인: 모두 `stopped`, 정상 종료 실패 시 원인 확인 후 별도 조치

다시 사용할 때 컨트롤 플레인부터 시작

```sh
qm start 910
qm start 911
qm start 912
```

VM 부팅 후 컨트롤 플레인에서 `kubectl get nodes`로 세 노드 Ready 확인, `kubeadm init`·`join` 재실행 불필요

## 실습 VM 전체 삭제

클러스터 데이터와 VM 디스크가 사라지는 되돌릴 수 없는 작업, 보관할 데이터가 없는 실습 VM에만 수행

**Proxmox에서**, 삭제 전 VM 이름과 번호 확인

```sh
qm list
qm config 910 | grep '^name:'
qm config 911 | grep '^name:'
qm config 912 | grep '^name:'
```

확인: `k8s-lab-cp01`, `k8s-lab-w01`, `k8s-lab-w02`, 다른 이름이면 삭제 중단

아직 실행 중이면 앞의 정상 종료 절차 사용, 세 VM 모두 `stopped` 확인 후 진행

```sh
qm set 910 --protection 0
qm set 911 --protection 0
qm set 912 --protection 0
qm destroy 911
qm destroy 912
qm destroy 910
qm list
```

템플릿 9001과 기존 VM 100·900 등은 삭제하지 않음

**Mac에서**, VM 삭제가 끝난 뒤 해당 호스트 키 기록 정리

```sh
ssh-keygen -R 10.10.10.110
ssh-keygen -R 10.10.10.111
ssh-keygen -R 10.10.10.112
```

## 다시 실습할 때

[VM 준비](01-prepare-vms.md)부터 새로 시작

`kubeadm reset`만으로 CNI 설정·네트워크 규칙·사용자 kubeconfig가 모두 정리되지는 않으므로, 이번 실습에서는 VM 삭제 후 재생성을 기본 방식으로 사용

## 운영 클러스터로 넘어가기 전

- [운영 구성 결정 항목](README.md#나중에-정할-운영-구성) 검토
- 데이터가 있는 VM과 클러스터는 실습 삭제 절차 적용 금지
- Kubernetes 지원 버전과 업그레이드 계획 확인
- 비밀정보 저장·주입 방식과 접근 권한 결정
- VM 백업과 별도로 etcd·영구 볼륨·앱 데이터의 백업·복원 계획 수립
- 구성·매니페스트는 Git, 토큰·kubeconfig·비밀키는 Git 밖에서 관리
- 실제 생성한 VM·버전·네트워크는 [서버 정보](../../reference/server.md)에 실측 기준으로 반영

참고: [kubeadm reset의 범위](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-reset/)
