k8s를 구성하는 방법 3가지
- 로컬 쿠버네티스
  - Minikube
  - Docker Desktop for Mac/Windows
  - Kind (k8s in docker)
- 쿠버네티스 구축 도구
  - kubeadm
  - Rancher
- 관리형 쿠버네티스 서비스
  - GKE
  - AKS
  - EKS
---
## 로컬 쿠버네티스

### minikube
minikube는 로컬 가상 머신에 쿠버네티스를 설치하기 위해 하이퍼바이저가 필요하다  
하이퍼바이저는 운영체제에 환경에 맞는 소프트웨어를 사전에 설치해야 한다

- mac os
  - hyperkit driver (Docker Desktop 에 포함됨)
  - virtualbox
  - parallels
  - vmware fusion
  - pod man...
- linux (생략)
- window (생략)


```shell
# Docker Desktop이 설치된 상태이기 때문에 생략
# brew install hyperkit

brew install minikube

minikube start driver=hyperkit

# minikube 클러스터 상태 확인
minikube status

# 컨텍스트 전환
kubectl config use-context minikube

# 노드 확인
kubectl get nodes

# minikube 클러스터 정지
minikube delete
```

### kind (kubernetes in docker)
도커 컨테이너를 여러 개 기동하고 그 컨테이너를 쿠버네티스 노드로 사용하여 여러 대로 구성된 쿠버네티스 클러스터를 구축한다
로컬 환경에서 멀티 노드 클러스터를 구성할 때 좋다

> 클러스터를 구축할 때는 Docker Desktop의 도커 환경이 기동되고 있어야 한다

```shell
brew install kind

kind create cluster --config kind.yaml --name kindcluster

# 클러스터 생성중 오류로 작업이 중단되는 현상 발생
# 도커 메모리 증설로 해결

kubectl config use-context kind-kindcluster

kubectl get nodes
# NAME                         STATUS   ROLES           AGE     VERSION
# kindcluster-control-plane    Ready    control-plane   3m9s    v1.24.1
# kindcluster-control-plane2   Ready    control-plane   2m50s   v1.24.1
# kindcluster-control-plane3   Ready    control-plane   111s    v1.24.1
# kindcluster-worker           Ready    <none>          105s    v1.24.1
# kindcluster-worker2          Ready    <none>          105s    v1.24.1
# kindcluster-worker3          Ready    <none>          105s    v1.24.1

kind delete cluster --name kindcluster
```


