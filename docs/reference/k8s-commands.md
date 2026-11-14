# Kubernetes 명령어

[기초 클러스터 실습](../tutorial/k8s-basics/README.md)에서 쓰는 명령 중심의 참고 목록, Kubernetes v1.35 기준

`kubectl`은 클러스터 자원 관리, `kubeadm`은 클러스터 초기화·가입 관리, `kubelet`은 각 노드의 에이전트

## 실행 위치와 공통 값

**Mac에서** 컨트롤 플레인 접속

```sh
ssh -J pve ubuntu@10.10.10.110
```

이하 `kubectl` 명령은 **컨트롤 플레인의 ubuntu 계정에서** 실행, [kubeconfig 설정](../tutorial/k8s-basics/03-create-cluster.md#3-ubuntu-계정의-kubectl-설정) 완료 필요

아래 변수를 현재 터미널에서 지정, Pod 이름은 조회 결과로 교체

```sh
NS=k8s-practice
DEPLOY=web
SERVICE=web
POD='<Pod-이름>'
```

`-n`은 namespace 지정, 생략하면 현재 context의 namespace 사용, 별도 지정이 없으면 `default`

목록의 명령은 필요한 것만 개별 사용, 위에서 아래로 모두 실행하는 설치 스크립트가 아님

## 클러스터와 접속 대상

```sh
kubectl config current-context             # 현재 접속 context
kubectl config get-contexts                # 등록한 context 목록
kubectl cluster-info                       # API 등 클러스터 접속 정보
kubectl version                            # 클라이언트와 서버 버전
kubectl get nodes -o wide                   # 노드 상태·IP·버전
kubectl get namespaces                     # namespace 목록
```

변경·삭제 명령 전 context 확인, kubeconfig와 `/etc/kubernetes/admin.conf`는 자격증명 포함으로 Git 저장 금지

## 자원 보기

```sh
kubectl get pods -A                                         # 모든 namespace의 Pod
kubectl get pods -n "$NS" -o wide                            # Pod·IP·배치 노드
kubectl get pods -n "$NS" -w                                 # Pod 상태 변화 관찰
kubectl get pods -n "$NS" -l app=web                          # 라벨로 필터
kubectl get deployments,replicasets,pods,services -n "$NS"    # 앱 관련 자원
kubectl get daemonsets -A                                   # 노드별 실행 자원
kubectl describe pod "$POD" -n "$NS"                         # Pod 상세 상태·이벤트
kubectl describe node k8s-lab-w01                            # 노드 상태·자원·이벤트
kubectl get pod "$POD" -n "$NS" -o yaml                       # Pod 객체를 YAML로 조회
```

`-w` 종료는 `Ctrl+C`, 조회 종료만 수행하며 Pod는 유지

## 매니페스트 적용

아래 `FILE`은 적용할 YAML 경로로 교체

```sh
FILE=~/k8s-practice-web.yaml
kubectl diff -f "$FILE"                    # 현재 상태와 적용할 내용 비교
kubectl apply --dry-run=server -f "$FILE"   # 저장 없이 API 검증
kubectl apply -f "$FILE"                   # 생성 또는 설정 반영
```

`diff`의 종료 코드 1은 차이 존재 의미, 반드시 실패를 뜻하지 않음

매니페스트에 `metadata.namespace`가 있으면 해당 namespace 확인 필요, dry-run은 앱 실행 성공까지 보장하지 않음

## 배포 상태와 변경

```sh
kubectl rollout status deployment/"$DEPLOY" -n "$NS" --timeout=180s   # 배포 완료 대기
kubectl rollout history deployment/"$DEPLOY" -n "$NS"                 # Pod 템플릿 변경 이력
kubectl rollout restart deployment/"$DEPLOY" -n "$NS"                 # Pod를 교체하는 재배포
kubectl scale deployment/"$DEPLOY" -n "$NS" --replicas=2               # 원하는 Pod 수 지정
kubectl rollout undo deployment/"$DEPLOY" -n "$NS"                    # 이전 Pod 템플릿으로 복귀
```

`undo`는 앱 데이터·Service·ConfigMap 등 모든 변경을 되돌리는 명령이 아님

실습 앱은 같은 노드에 웹 Pod를 중복 배치하지 않는 설정, 워커 두 대에서 replicas를 3으로 늘리면 일부 Pod `Pending` 가능

매니페스트를 다시 적용하면 명령으로 바꾼 replicas 등이 파일 내용으로 바뀔 수 있음

## 로그와 컨테이너 명령

```sh
kubectl logs "$POD" -n "$NS" --tail=100           # 최근 로그
kubectl logs "$POD" -n "$NS" -f                  # 로그 실시간 확인
kubectl logs "$POD" -n "$NS" --previous          # 직전 종료 컨테이너 로그
kubectl exec "$POD" -n "$NS" -- hostname         # 컨테이너 안에서 명령 실행
kubectl exec -it "$POD" -n "$NS" -- sh            # 컨테이너 셸 접속
```

여러 컨테이너가 있으면 `-c <컨테이너-이름>` 추가, `--previous`는 이전 컨테이너 기록이 있을 때 사용 가능

`exec`의 `--` 뒤는 컨테이너 내부 명령, 이미지에 해당 실행 파일이 있어야 함

## Service와 통신 확인

```sh
kubectl get services -n "$NS"                                    # ClusterIP와 포트
kubectl get endpointslices -n "$NS" -l kubernetes.io/service-name=web   # Service의 대상 Pod 주소
kubectl exec netcheck -n "$NS" -- nslookup web.k8s-practice.svc.cluster.local   # DNS 확인
kubectl exec netcheck -n "$NS" -- wget -T 5 -qO- http://web         # Service HTTP 확인
kubectl port-forward -n "$NS" --address=127.0.0.1 service/"$SERVICE" 8080:80   # 로컬 포트로 앱 연결
```

`netcheck`는 [앱 배포 실습](../tutorial/k8s-basics/04-deploy-and-check.md#2-워커-1에-통신-확인용-pod-생성)에서 생성한 진단 Pod, 존재하고 실행 중일 때 사용

컨트롤 플레인에서 연 포트는 Mac의 localhost와 다름, Mac 접속은 [SSH 터널 절차](../tutorial/k8s-basics/04-deploy-and-check.md#5-mac에서-웹-응답-확인) 사용

port-forward는 선택한 Pod로 연결하는 진단 경로, Service의 데이터 경로·분산 동작 검증을 대신하지 않음

## 삭제

대상과 namespace 확인 후 필요한 명령만 실행

```sh
kubectl delete pod "$POD" -n "$NS"                    # Pod 한 개 삭제
kubectl delete deployment "$DEPLOY" -n "$NS"          # Deployment와 관리하는 Pod 삭제
kubectl delete service "$SERVICE" -n "$NS"            # Service 삭제
kubectl delete -f "$FILE"                             # 파일에 지정한 자원 삭제
kubectl delete namespace "$NS"                       # namespace와 그 안의 자원 전체 삭제
```

Deployment가 관리하는 Pod만 삭제하면 컨트롤러가 원하는 수를 유지하도록 새 Pod 생성

Deployment 삭제만으로 별도 Service까지 삭제되지는 않음, namespace 삭제는 해당 namespace 전체에 영향

## 문제가 생겼을 때

**컨트롤 플레인에서**

```sh
kubectl get events -n "$NS" --sort-by=.metadata.creationTimestamp   # 최근 앱 이벤트
kubectl get events -A --sort-by=.metadata.creationTimestamp         # 전체 이벤트
kubectl get pods -n kube-system -o wide                            # 시스템 Pod
kubectl get pods -n kube-flannel -o wide                           # Flannel Pod
kubectl logs -n kube-flannel -l app=flannel -c kube-flannel --tail=100   # Flannel 로그
kubectl explain deployment.spec.replicas                          # 필드 설명
kubectl api-resources                                             # 지원하는 자원 종류
kubectl get --help                                                # 명령별 도움말
```

기본 확인 순서: `get`으로 대상 찾기 → `describe`로 상태·이벤트 확인 → `logs`로 앱 오류 확인

## kubeadm과 노드 서비스

**컨트롤 플레인에서**

```sh
kubeadm version -o short                              # kubeadm 버전
sudo kubeadm token list                               # 가입 토큰 목록
sudo kubeadm token create --ttl 30m --print-join-command   # 30분 유효 가입 명령 생성
```

필요 없는 토큰은 `sudo kubeadm token delete <TOKEN-ID>`로 삭제, 토큰·가입 명령 출력은 공개 로그와 Git 저장 금지

`init`·`join`은 실행 위치와 네트워크 설정이 중요하므로 [클러스터 생성 절차](../tutorial/k8s-basics/03-create-cluster.md) 사용

**문제가 있는 VM에서**, 해당 VM에 SSH 접속 후 실행

```sh
systemctl is-active kubelet containerd                 # 노드 서비스 동작 확인
sudo journalctl -u kubelet -n 100 --no-pager             # kubelet 로그
sudo journalctl -u containerd -n 100 --no-pager           # 런타임 로그
```

Proxmox 호스트에서는 위 노드 서비스를 진단하지 않음, `kubeadm reset`은 일반 재시작 명령이 아니므로 [정리 절차](../tutorial/k8s-basics/05-cleanup.md) 확인

## 공식 참고

- [kubectl 명령 구조](https://kubernetes.io/docs/reference/kubectl/)
- [kubectl 명령 상세](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands)
- [kubectl 빠른 참고](https://kubernetes.io/docs/reference/kubectl/quick-reference/)
- [kubeadm 가입 토큰](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-token/)
