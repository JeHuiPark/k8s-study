# 쿠버네티스 아키텍처
쿠버네티스 클러스터는 8가지 요소로 구성되며 kube-apiserver를 중심으로 분산 시스템으로 되어 있다

- etcd
- kube-apiserver
- kube-scheduler
- kube-controller-manager
- kubelet(+컨테이너 런타임)
- kube-proxy
- cni plugin
- kube-dns(CoreDns)
- \[option\] cloud-controller-manager

## etcd
쿠버네티스 클러스터에 등록된 모든 정보가 저장되어 있음

## kube-apiserver
쿠버네티스 API를 제공하는 구성 요소

## kube-scheduler
기동할 노드 정보가 할당되어 있지 않은 파드를 감지하여 kube-apiserver에 요청을 보내고 업데이트함으로써 스케줄링한다.  
이때 kube-scheduler는 각 쿠버네티스 노드 상태, 노드 어피니티나 인터파드 어피니티 등의 조건에 부합하는지를 판단하여 기동할 노드를 결정한다.

## kube-controller-manager
다양한 컨트롤러를 실행하는 구성 요소  
디플로이먼트 컨트롤러나 레플리카셋 컨트롤러에는 디플로이먼트나 레플리카셋 상태를 모니터링하면서 필요에 따라 레플리카셋이나 파드를 생성한다.  
kube-controller-manager는 파드만 등록하고, 실제 스케줄링은 kube-scheduler에서 진행한다.

노드 컨트롤러, 서비스 어카운트, 토큰 컨트롤러...

## kubelet
각 쿠버네티스 노드에서 동작하는 구성요소  
도커 등의 컨테이너 런타임과 연계하여 실제 컨테이너에 대한 기동과 정지 등을 관리
> kube-apiserver를 통해 파드가 등록된 후 kube-scheduler에 의해 해당 파드가 기동되는 노드가 결정되면 kubelet은 그것을 감지하고 자신의 노드에서 기동해야 하는 컨테이너를 기동한다


## kube-proxy
각 쿠버네티스 노드에서 동작하는 구성요소  

서비스 리소스가 생성되었을 때 ClusterIP나 NodePort로 가는 트래픽이 파드에 정상적으로 전송되도록 한다.

3가지 전송 방식이 있다
- userspace
- iptables
- ipvs

## cni plugin

쿠버네티스는 별도 오버레이 네트워크를 구축하는 소프트웨어와 사용해야 한다.  
쿠버네티스 클러스터는 여러 쿠버네티스 노드로 구성되어 있다.  
여러 쿠버네티스 노드는 파드 간 통신을 확보하기 위해 클러스터 내에 분산 배치된 파드가 서로 통신이 가능하도록 네트워크를 구성해야 하는데, **cni plugin이 이것을 담당한다**

## kube-dns(CoreDNS)
쿠버네티스 클러스터 내부의 이름 해석이나 서비스 디스커버리에 사용되는 클러스터 내부의 DNS 서버

## cloud-controller-manager
쿠버네티스 클러스터가 각종 클라우드 프로바이더와 연계하기 위한 구성요소