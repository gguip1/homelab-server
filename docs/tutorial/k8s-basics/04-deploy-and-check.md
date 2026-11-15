# 4. 앱 배포와 통신 확인

시작: 노드 세 대 모두 `Ready`

Mac에서 워커 IP나 Pod IP로 직접 접근하는 대신, 먼저 클러스터 내부 통신 확인 후 SSH 터널로 앱 확인

## 1. 서로 다른 워커에 웹 Pod 배포

**컨트롤 플레인의 ubuntu 계정에서**

```sh
kubectl create namespace k8s-practice
cat > ~/k8s-practice-web.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: k8s-practice
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: web
              topologyKey: kubernetes.io/hostname
      containers:
        - name: nginx
          image: nginx:1.28-alpine
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 50m
              memory: 32Mi
            limits:
              memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: k8s-practice
spec:
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
EOF
kubectl apply -f ~/k8s-practice-web.yaml
kubectl -n k8s-practice rollout status deployment/web --timeout=180s
kubectl -n k8s-practice get pods -o wide
kubectl -n k8s-practice get service web
```

확인: 웹 Pod 두 개의 `READY`가 `1/1`, 서로 다른 워커에 배치, Service에 ClusterIP 할당

podAntiAffinity는 같은 앱의 Pod를 같은 노드에 배치하지 않는 설정, 워커 한 대가 사용 불가능하면 두 번째 웹 Pod는 `Pending` 가능

Deployment는 Pod 수 유지, Service는 바뀌는 Pod 주소 앞에 고정 접점 제공

## 2. 워커 1에 통신 확인용 Pod 생성

**컨트롤 플레인에서**

```sh
kubectl -n k8s-practice run netcheck --image=busybox:1.37 \
  --restart=Never \
  --overrides='{"spec":{"nodeName":"k8s-lab-w01"}}' \
  --command -- sleep 3600
kubectl -n k8s-practice wait --for=condition=Ready pod/netcheck --timeout=180s
kubectl -n k8s-practice get pods -o wide
```

확인: netcheck가 `k8s-lab-w01`에서 실행, 진단용으로 한 시간 후 종료되는 Pod

## 3. DNS와 Service 통신

```sh
kubectl -n k8s-practice exec netcheck -- nslookup web.k8s-practice.svc.cluster.local
kubectl -n k8s-practice exec netcheck -- wget -T 5 -qO- http://web
```

확인: DNS에서 Service ClusterIP 반환, HTTP 응답에 `Welcome to nginx!` 포함

## 4. 두 웹 Pod로 직접 통신

```sh
for pod_ip in $(kubectl -n k8s-practice get pods -l app=web -o jsonpath='{.items[*].status.podIP}'); do
  printf '\nPod IP: %s\n' "$pod_ip"
  kubectl -n k8s-practice exec netcheck -- wget -T 5 -qO- "http://$pod_ip"
done
```

확인: 두 Pod IP 모두 nginx 응답, 그중 워커 2의 Pod 응답으로 노드 사이 Pod 통신 확인

Service 요청 한 번만으로 다른 노드의 Pod 통신 성공 여부를 판단하지 않음

## 5. Mac에서 웹 응답 확인

**Mac 터미널 A에서** SSH 로컬 포트포워딩과 함께 컨트롤 플레인 접속

```sh
ssh -o ExitOnForwardFailure=yes -J pve \
  -L 127.0.0.1:18080:127.0.0.1:8080 ubuntu@10.10.10.110
```

접속된 **컨트롤 플레인에서**

```sh
kubectl -n k8s-practice port-forward --address=127.0.0.1 service/web 8080:80
```

확인: `Forwarding from 127.0.0.1:8080 -> 80`, 이 터미널은 실행 상태 유지

**Mac 터미널 B에서**

```sh
curl http://127.0.0.1:18080
```

브라우저에서도 `http://127.0.0.1:18080` 열기 가능

접속 경로: Mac 18080 → SSH 터널 → 컨트롤 플레인 8080 → kubectl port-forward → 웹 Pod 80

이 방식은 선택한 Pod로 터널 연결하는 확인용 경로, Ingress·LoadBalancer 또는 Service 데이터 경로의 검증을 대신하지 않음

종료: 터미널 A에서 `Ctrl+C` 후 `exit`

## 6. Pod 재생성 확인

**컨트롤 플레인에서**, port-forward 종료 후 실행

```sh
kubectl -n k8s-practice get pods -l app=web
WEB_POD=$(kubectl -n k8s-practice get pods -l app=web -o jsonpath='{.items[0].metadata.name}')
kubectl -n k8s-practice delete pod "$WEB_POD"
kubectl -n k8s-practice get pods -l app=web -w
```

새 이름의 Pod가 생성되고 웹 Pod 두 개가 다시 `Running`, `READY 1/1`이면 `Ctrl+C`

Pod 삭제와 VM 장애는 다른 실습, 이 결과만으로 컨트롤 플레인 HA나 저장 데이터 복구를 보장하지 않음

## 완료 기준과 문제 확인

- 노드 세 대 `Ready`
- Flannel과 CoreDNS 정상
- 웹 Pod 두 개가 서로 다른 워커에서 실행
- DNS·Service·각 Pod IP 요청 성공
- Mac 브라우저 또는 curl에서 nginx 응답
- 웹 Pod 삭제 후 자동 재생성

실패 시 컨트롤 플레인에서 확인

```sh
kubectl -n k8s-practice get pods -o wide
kubectl -n k8s-practice get events --sort-by=.metadata.creationTimestamp
kubectl -n k8s-practice describe deployment web
kubectl -n k8s-practice get endpointslices -l kubernetes.io/service-name=web
```

| 증상 | 확인 |
|---|---|
| ImagePullBackOff | 노드의 DNS·인터넷·이미지 저장소 접근 |
| 웹 Pod Pending | 워커 두 대 Ready 여부, 자원, anti-affinity 이벤트 |
| DNS만 실패 | CoreDNS·Service·kube-proxy 상태와 방화벽 |
| 다른 워커 Pod만 timeout | Flannel·UDP 8472·forwarding·MTU |
| Mac에서 connection refused | 터널과 port-forward 두 단계가 모두 실행 중인지 확인 |

다음: [종료와 정리](05-cleanup.md)

참고: [Pod 배치](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/), [Service](https://kubernetes.io/docs/concepts/services-networking/service/), [포트포워딩](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/)
