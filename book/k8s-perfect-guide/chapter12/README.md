# 스케쥴링

## 매니페스트에서 지정하는 스케쥴링

[쿠버네티스 스케쥴링에는 '필터링' 과정과 '스코어링' 과정이 있다.](https://kubernetes.io/ko/docs/reference/scheduling/policies/)

**쿠버네티스의 스케쥴링 분류**
- 쿠버네티스 사용자가 배치하고 싶은 노드를 선택하는 방법
  - nodeSelector (filtering)
  - Node Affinity (flitering, scoring)
  - Node Anti-Affinity (flitering, scoring)
  - Inter-Pod Affinity (flitering, scoring)
  - Inter-Pod Anti-Affinity (flitering, scoring)
- 쿠버네티스 관리자가 배치하고 싶지 않은 노드를 지정하는 방법
  - Taints / Tolerations (flitering, scoring)

| 종류                      | 개요                     |
|-------------------------|------------------------|
| nodeSelector            | 단순한 노드 어피니티            |
| Node Affinity           | 특정 노드상에서만 실행           |
| Node Anti-Affinity      | 특정 노드 이외에서 실행          |
| Inter-Pod Affinity      | 특정 파드가 존재하는 도메인에서 실행   |
| Inter-Pod Anti-Affinity | 특정파드가 존재하지 않는 도메인에서 실행 |


## 빌트인 노드 레이블과 레이블 추가

쿠베네티스 사양에서는 빌트인 노드 레이블에 최소한 호스트명/OS/아키텍처/인스턴스 타입/리전/존과 같은 정보를 부여하게 되어 있다.  
환경에 따라 쿠버네티스 플롯팸이나 분산 환경 자체의 레이블도 부여된다

```shell
# 노드에 할당된 레이블 정보
kubectl get nodes -o json | jq '.items[] | .metadata.labels'
# {
#   "beta.kubernetes.io/arch": "arm64",
#   "beta.kubernetes.io/os": "linux",
#   "kubernetes.io/arch": "arm64",
#   "kubernetes.io/hostname": "minikube",
#   "kubernetes.io/os": "linux",
#   "minikube.k8s.io/commit": "362d5fdc0a3dbee389b3d3f1034e8023e72bd3a7",
#   "minikube.k8s.io/name": "minikube",
#   "minikube.k8s.io/primary": "true",
#   "minikube.k8s.io/updated_at": "2022_06_18T20_23_14_0700",
#   "minikube.k8s.io/version": "v1.25.2",
#   "node-role.kubernetes.io/control-plane": "",
#   "node-role.kubernetes.io/master": "",
#   "node.kubernetes.io/exclude-from-external-load-balancers": ""
# }

# 노드 레이블 지정
kubectl label node minikube disktype=ssd
kubectl label node minikube-m02 disktype=ssd
kubectl label node minikube-m03 disktype=hdd

# 노드 목록과 disktype 레이블 표시
kubectl -L=disktype,another-label get node
```

## NodeSelector
단순한 노드 어피니티만을 실행하는 경우 NodeSelector를 사용할 수 있다

```shell
kubectl apply -f sample-nodeselector.yaml # disktype=ssd 라벨링이 된 2번 노드에 배치
kubectl apply -f sample-nodeselector-fail.yaml # 조건에 맞는 노드가 없어서 펜딩

kubectl get po -o wide
# NAME                       READY   STATUS    RESTARTS   AGE   IP           NODE           NOMINATED NODE   READINESS GATES
# sample-nodeselector        1/1     Running   0          19s   10.244.1.2   minikube-m02   <none>           <none>
# sample-nodeselector-fail   0/1     Pending   0          19s   <none>       <none>         <none>           <none>
```

## Node Affinity

Node Affinity는 파드를 특정 노드에 스케줄링하는 정책이다.  
`spec.affinity.nodeAffinity`에 작성하여 노드 어피니티를 설정한다.

Node Affinity의 필수 스케줄링 정책과 우선 스케줄링 정책

| 설정 항목                                           | 개요                 |
|-------------------------------------------------|--------------------|
| requiredDuringSchedulingIgnoredDuringExecution  | 필수 스케줄링 정책         |
| preferredDuringSchedulingIgnoredDuringExecution | 우선적으로 고려되는 스케줄링 정책 |

```shell
kubectl apply -f sample-node-affinity.yaml
# minikube-m02 노드에 생성됨
kubectl get po sample-node-affinity -o wide

kubectl delete po sample-node-affinity

# 우선 조건으로 지정된 노드를 unscheduling 상태로 변경
kubectl drain minikube-m02 --ignore-daemonsets
kubectl apply -f sample-node-affinity.yaml
# minikube 노드에 생성됨
kubectl get po sample-node-affinity -o wide

# minikube-m02 노드를 스케쥴링 대상으로 변경
kubectl uncordon minikube-m02
```

파드 생성시 **필수 조건**에 만족하는 노드가 없으면 pending 상태가 된다

## Node Anti-Affinity

`spec.affinity.nodeAffinity` 에 부정형 오퍼레이터를 지정하여 설정

## Inter-Pod Affinity

인터파드 어피니티는 특정 파드가 실행되는 도메인(노드, 존 등)에 파드를 스케줄링하는 정책이다.  
`spec.affinity.podAffinity`에 조건을 지정하여 파드 어피니티를 설정하고, 파드끼리는 가까이 배치할 수 있으므로 파드 간의 통신 레이턴시를 낮출 수 있다

`topologyKey`는 어느 범위를 스케줄링 대상으로 할지를 지정한다.  
`sample-pod-affinity-host.yaml` 매니페스트를 해석하면 아래와 같다
- app=sample-app 레이블을 가진 파드가 실행되는 노드이면서
- 같은 topologyKey 를 갖는 노드에 
- sample-pod-affinity-host 파드를 배포한다 

## Inter-Pod Anti-Affinity

`spec.affinity.podAntiAffinity`를 지정하여 설정한다