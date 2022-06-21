# 리소스 제한

쿠버네티스는 컨테이너 단위로 리소스 제한 설정이 가능하다
- cpu
- 메모리
- Ephemeral 스토리지
- \+ Device Plugins을 사용하여 다른 리소스

## CPU/메모리 리소스 제한

CPI 는 1vCPU를 1000m(millicores) 단위로 지정한다.
> 3GHz CPU라면 1코어를 지정할 경우 1000m가 된다

| 리소스 유형 | 단위                        |
|--------|---------------------------|
| CPU    | 1 = 1000m = 1 vCPU        |
| 메모리    | 1G = 1000M (1Gi = 1024Mi) |

리소스 제한은 파드 정의 내부의 각 컨테이너 정의 부분에 포함됨
- `spec.containers[].resources`
  - `request`
    - `cpu`
    - `memory`
  - `limits`
    - `cpu`
    - `memory`

`requests`는 사용하는 리소스 최솟값을 지정한다. 그래서 빈 노드에 `requests`로 지정한 양의 리소스가 존재하지 않으면 스케쥴링되지 않는다.  
`limits`는 사용할 리소스 최댓값을 지정한다. `requests`와 달리 노드에 `limits`로 지정한 리소스가 남아 있지 않아도 스케쥴링된다

```shell
# 노드에 할당된 리소스 확인 Allocated resources:
kubectl describe node minikube

kubectl apply -f sample-resource.yaml

# 노드 리소스 변화 확인 Allocated resources:
kubectl describe node minikube
```

`requests`만 설정한 경우 `limits` 을 생략하면 OOM으로 인해 프로세스가 정지될 수 있다 (명시적으로 둘 다 설정필요)  
반대로 `limits`만 설정하면 `requests`의 값이 `limits`값으로 설정 됨

```shell
kubectl apply -f sample-resource-only-limit.yaml
kubectl get po sample-resouce-only-limit  -o json | jq ".spec.containers[].resources"
```
