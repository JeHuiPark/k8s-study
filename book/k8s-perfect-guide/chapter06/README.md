# 서비스 API 카테고리

## ClusterIP

- 로드 밸런서
- 클러스터 내부에서만 접속 가능
- ExternalIP 옵션으로 특정 쿠버네티스 노드로 들어오는 요청을 수신할 수 있음 (외부 수신)

```shell
kubectl apply -f sample-deployment.yaml
kubectl apply -f sample-cluster-ip.yaml

# Pod internal IP 확인 
kubectl get po -l app=sample-app -o custom-columns="NAME:{metadata.name}, IP:{status.podIP}"

# ClusterIP 에 연결된 엔드포인트 확인
kubectl describe svc sample-cluster-ip | grep -i Endpoints
```

## NodePort

모든 쿠버네티스 노드에서 수신한 요청을 컨테이너에 전송하는 형태로 외부와 통신이 가능하게 한다  
0.0.0.0:포트를 사용하여 모든 IP 주소로 바인드하는 형태

```shell
kubectl apply -f sample-nodeport.yaml
kubectl get svc
```

## LoadBalancer

- 쿠버네티스 클러스터 외부의 로드 밸런서에 외부 통신이 가능한 가상ip를 할당
- 쿠버티스 노드와 별도로 외부 로드 밸런서를 사용하기 때문에 노드 장애와 로드 밸런서와 격리
> NodePort, ExternalIP 에서는 쿠버네티스 노드에 할당된 IP 주소로 통신을 하기 때문에 노드가 SPoF가 된다

## ExternalName
서비스명으로 임의의 CNAME을 반환

## HeadlessService

- HeadlessService는 대상이 되는 개별 파드의 IP 주소가 직접 반환된다
- 로드밸런싱을 위한 엔드포인트(IP 주소)가 제공되지 않는다
- DNS 라운드 로빈을 사용한 엔드포인트를 제공한다
  - 클러스터 내부 DNS 에서 반환되는 형태로 부하 분산을 하기 때문에 클라이언트에서 DNS 캐시에 주의해야 한다

```shell
kubectl apply -f sample-headless.yaml


kubectl run --image=amsy810/tools:v2.0 --restart=Never --rm -i testpod \ 
--command -- dig sample-headless.default.svc.cluster.local

kubectl run --image=amsy810/tools:v2.0 --restart=Never --rm -i testpod \
--command -- dig sample-stateful-set-headless-0.sample-headless.default.svc.cluster.local

kubectl apply -f sample-subdomain.yaml

kubectl run --image=amsy810/tools:v2.0 --restart=Never --rm -i testpod \
--command -- dig sample-hostname.sample-subdomain.default.svc.cluster.local
```