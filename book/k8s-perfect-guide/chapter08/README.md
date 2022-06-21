# 클러스터 & 메타데이터 카테고리

## 노드

노드 리소스는 `status.allocatable`과 `status.capacity`에서 확인할 수 있다.  
capcity는 해당 노드가 소유하고 있는 cpu나 메모리의 실제 용량이다. allocatable은 쿠버네티스가 시스템 리소스로 할당한 만큼 뺀, 즉 실제 파드에 할당 가능한 리소스 용량이다.  
할당할 수 있는 남은 리소스 양을 확인하려면 allocatable에서 현재 리소스 사용량을 빼야 한다.

## 네임스페이스

```shell
kubectl apply -f sample-namespace.yaml
kubectl create namespace sample-namespace

# sample-namespace 네임스페이스의 파드 목록 조회
kubectl get po -n sample-namespace

# 모든 네임스페이스의 파드 목록 조회
kubectl get po -A
```