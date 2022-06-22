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

## 시스템에 할당된 리소스와 Eviction 매니저
cpu/memory/ephemeral 스토리지의 일반적인 리소스는 완전히 고갈되면 쿠버네티스 자체가 동작하지 않거나 그 노드 전체에 영향을 미칠 수 있다.  
따라서 각 노드에는 kube-reserved, system-reserved라는 두 가지 리소스가 시스템용으로 확보되어 있다

쿠버네티스 내부에서는 Eviction 매니저라는 구성 요소가 동작하며 시스템 전체가 과부화되지 않도록 관리한다.  
Eviction 매니저는 실제로 사용되는 리소스 합계가 Eviction Threshold를 초과하지 않는지 정기적으로 확인하고, 초과한 경우 파드를 Evict한다  (OOM 발생 제어)

| Eviction Threshold | desc                |
|--------------------|---------------------|
| soft               | sigterm 신호를 보내 파드정지 |
| hard               | sigkill 신호를 보내 파드정지 |

## 오버커밋과 리소스 부족

```shell
# 스케일 아웃
kubectl scale deployment --replicas 24 sample-resource

# pending 상태 확인
kubectl get po

# deployment 상태 확인
kubectl get deployment

# 노드 리소스 상태 확인
kubectl describe node minikube
# Allocated resources:
#   (Total limits may be over 100 percent, i.e., overcommitted.)
#  Resource           Requests      Limits
#  --------           --------      ------
#  cpu                2350m (58%)   2500m (62%)
#  memory             3332Mi (84%)  5290Mi (134%)
```

memory requests 제한이 16% 남음(pod의 memory request 값 1024Mi를 허용할 리소스가 없음)

```shell
# pending 상태의 파드 확인 
kubectl describe po sample-resource-57686756db-25jl6
# Events:
#   Type     Reason            Age                From               Message
#   ----     ------            ----               ----               -------
#   Warning  FailedScheduling  25m                default-scheduler  0/1 nodes are available: 1 Insufficient memory.
#   Warning  FailedScheduling  22m (x1 over 23m)  default-scheduler  0/1 nodes are available: 1 Insufficient memory.
```

여기에서 파드는 'request 리소스 양 < limit 리소스 양'으로 설정되어 있음. 이 상태에서 파드에 부하가 증가하면 노드 리소스 사용량이 100%를 초과하더라도 오버커밋하여 실행됨
> OOM 킬러로 인해 다른 프로세스가 종료될 수 있음

기본 정책은 Requests와 Limits에 너무 큰 차이를 주지 않을 것

## LimitRange를 사용한 리소스 제한

LimitRanage를 사용하면 파드 등에 대해 CPU나 메모리 리소스의 최솟값과 최댓값, 기본값 등을 설정할 수 있다.  
네임스페이스별 설정도 가능하다.  
기존 파드에는 영향을 주지 않는다.

| 설정 항목                | 개요                 |
|----------------------|--------------------|
| default              | 기본 Limit           |
| defaultRequest       | 기본 Requests        |
| max                  | 최대 리소스             |
| min                  | 최소 리소스             |
| maxLimitRequestRatio | Limits/Requests 비율 |

| 타입        | 사용가능한 설정 항목                  |
|-----------|------------------------------|
| 컨테이너      | all                          |
| 파드        | max/min/maxLimitRequestRatio |
| 영구 볼륨 클레임 | max/min                      |

환경에 따라 기본 LimitRange 값이 설정되어 있는 경우도 있다

```shell
kubectl apply -f sample-pod.yaml

kubectl get po sample-pod -o json | jq ".spec.containers[].resources"
#{
#  "limits": {
#    "cpu": "500m",
#    "memory": "512Mi"
#  },
#  "requests": {
#    "cpu": "250m",
#    "memory": "256Mi"
#  }
#}

# LimitRange 에서 설정한 값보다 작게 요청하면
kubectl apply -f sample-pod-under-request.yaml

#  LimitRange 에서 설정한 request/limit 비율을 초과할 경우
kubectl apply -f sample-pod-over-ratio.yaml
```
