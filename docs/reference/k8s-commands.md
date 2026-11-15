# Kubernetes 명령어

클러스터 관리 명령, 컨트롤 플레인의 ubuntu 계정에서 실행, `<이름>`은 자원 이름

특정 namespace를 지정하려면 `-n <ns>`, 전체 namespace 조회는 `-A` 추가

## 보기

```sh
kubectl get nodes -o wide                  # 노드 상태·IP
kubectl get pods -o wide                   # Pod 상태·IP·배치 노드
kubectl get pods -w                        # Pod 상태 변화 (나가기: Ctrl+C)
kubectl get deployments                    # Deployment 목록
kubectl get replicasets                    # ReplicaSet 목록
kubectl get services                       # Service 목록
kubectl get namespaces                     # namespace 목록
kubectl describe pod <이름>                 # Pod 상세·이벤트
kubectl describe node <이름>                # 노드 상세
kubectl get pod <이름> -o yaml              # YAML로 조회
```

## 적용 / 삭제

```sh
kubectl apply -f <파일>                     # YAML 생성·반영
kubectl diff -f <파일>                      # 현재 상태와 YAML 비교
kubectl delete -f <파일>                    # YAML에 지정한 자원 삭제
kubectl delete pod <이름>                   # Pod 삭제
kubectl delete deployment <이름>            # Deployment와 관리하는 Pod 삭제
kubectl create namespace <ns>               # namespace 생성
kubectl delete namespace <ns>               # namespace 안의 자원까지 삭제
```

Deployment가 관리하는 Pod만 삭제하면 새 Pod 생성, Deployment 삭제만으로 별도 Service까지 삭제되지는 않음

## 배포 변경

```sh
kubectl scale deployment <이름> --replicas=2   # Pod 수 변경
kubectl rollout status deployment/<이름>       # 배포 진행 확인
kubectl rollout restart deployment/<이름>      # Pod 교체 재배포
kubectl rollout history deployment/<이름>      # Pod 템플릿 변경 이력
kubectl rollout undo deployment/<이름>         # 이전 Pod 템플릿으로 복귀
```

## 로그 / 컨테이너 접속

```sh
kubectl logs <Pod> --tail=100               # 최근 로그
kubectl logs <Pod> -f                       # 로그 실시간 확인 (나가기: Ctrl+C)
kubectl logs <Pod> --previous               # 직전 종료 컨테이너 로그
kubectl exec <Pod> -- <명령>                # 컨테이너 안에서 명령 실행
kubectl exec -it <Pod> -- sh                # 컨테이너 셸 접속
```

컨테이너가 여러 개면 `-c <컨테이너>` 추가

## Service / 통신

```sh
kubectl get services -o wide                # Service IP·포트
kubectl get endpointslices                  # Service 대상 주소
kubectl exec <Pod> -- nslookup <Service>    # DNS 확인 (nslookup 필요)
kubectl exec <Pod> -- wget -qO- http://<Service>   # HTTP 확인 (wget 필요)
kubectl port-forward service/<이름> 8080:80 --address=127.0.0.1   # 실행한 기기의 8080으로 연결
```

Mac에서 접속하려면 [SSH 터널](../tutorial/k8s-basics/04-deploy-and-check.md#5-mac에서-웹-응답-확인) 사용

## 문제가 생겼을 때

```sh
kubectl get events --sort-by=.metadata.creationTimestamp   # 이벤트 시간순
kubectl get pods -n kube-system -o wide                    # 시스템 Pod
kubectl get pods -n kube-flannel -o wide                   # Flannel Pod
kubectl explain deployment.spec.replicas                   # 필드 설명
kubectl config current-context                            # 현재 접속 대상
kubectl <명령> --help                                      # 명령 상세 설명
```

## kubeadm

컨트롤 플레인에서 실행, 가입 토큰과 kubeconfig는 Git 저장 금지

```sh
kubeadm version -o short                                   # 버전
sudo kubeadm token list                                    # 가입 토큰 목록
sudo kubeadm token create --ttl 30m --print-join-command    # 30분 유효 가입 명령 생성
sudo kubeadm token delete <TOKEN-ID>                       # 가입 토큰 삭제
```

초기화·가입은 [클러스터 생성](../tutorial/k8s-basics/03-create-cluster.md), 전체 정리는 [종료와 정리](../tutorial/k8s-basics/05-cleanup.md) 참고

## 노드 서비스

문제가 있는 VM에서 실행

```sh
systemctl is-active kubelet containerd                # 노드 서비스 상태
sudo journalctl -u kubelet -n 100 --no-pager            # kubelet 로그
sudo journalctl -u containerd -n 100 --no-pager          # 런타임 로그
```

공식 참고: [kubectl](https://kubernetes.io/docs/reference/kubectl/), [kubeadm token](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-token/)
