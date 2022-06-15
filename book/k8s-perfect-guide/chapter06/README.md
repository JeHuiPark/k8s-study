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
