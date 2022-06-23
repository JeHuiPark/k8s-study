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

## QoS Class

파드에는 Request/Limits 설정에 따라 자동으로 QoS Class 값이 설정되게 되어 있다.

| QoS Class  | 조건                                                   | 우선순위 |
|------------|------------------------------------------------------|------|
| BestEffort | Request/Limits 모두 미지정                                | 3    |
| Guaranteed | Requests/Limits 가 같고 cpu, memory 명시적 지정              | 1    |
| Burstable  | Guaranteed를 충족하지 못하고 한 개 이상의 Request/Limits가 설정되어 있음 | 2    |

QoS Class는 쿠버네티스가 컨테이너에 oom score를 설정할 때 사용된다 

| QoS Class  | 조건                                                       |
|------------|----------------------------------------------------------|
| BestEffort | 1000                                                     |
| Guaranteed | -998                                                     |
| Burstable  | min(max2, 1000 - (1000*메모리의 requests) / 머신 메모리 용량), 999) |

**oom score?**  
OOM Killer에 의해 프로세스를 정지시킬 때 우선순위 값으로 -1000~1000 범위에서 설정한다  
메모리 사용량이 노드 최댓값을 넘어 OOM Killer에 의해 컨테이너를 정지시킬 때 BestEffort, Burstable, Guaranteed 순서로 정지한다.  
Guaranteed의 경우에는 쿠버네티스 시스템 구성요소외에 우선순위가 높은 컨테이너가 없어서 좀 더 안정적인 시행이 가능하다.

## ResourceQuota

리소스쿼터를 사용하여 네임스페이스 단위로 사용 가능한 리소스를 제한할 수 있다  
_이미 생성된 리소스에는 영향을 주지 않는다_

```shell
kubectl apply -f sample-resourcequota.yaml
kubectl describe resourcequota
# Name:             sample-resourcequota
# Namespace:        default
# Resource          Used  Hard
# --------          ----  ----
# count/configmaps  1     10
for i in `seq 1 10`; do kubectl create configmap conf-$i --from-literal=key1=val1;done
# configmap/conf-1 created
# configmap/conf-2 created
# configmap/conf-3 created
# configmap/conf-4 created
# configmap/conf-5 created
# configmap/conf-6 created
# configmap/conf-7 created
# configmap/conf-8 created
# configmap/conf-9 created
# error: failed to create configmap: configmaps "conf-10" is forbidden: exceeded quota: sample-resourcequota, requested: count/configmaps=1, used: count/configmaps=10, limited: count/configmaps=10
```

리소스 쿼터가 설정되면 제한이 걸린 항목 설정은 필수로 입력해야 한다 (cpu, memory)

## HorizontalPodAutoscaler
HorizontalPodAutoscaler(HPA)는 디플로이먼트/레플리카셋/레플리케이션 컨트롤러의 레플리카 수를 CPU 부하 등에 따라 자동으로 스케일하는 리소스다.  
부하가 높아지면 스케일 아웃하고, 부하가 낮아지면 스케일 인된다.  
_파드에 Resource Requests가 설정되어 있지 않은 경우에는 동작하지 않는다._

- 일정 시간마다 오토 스케일링 여부를 확인 (필요한 레플리카 수 계산)
  - ceil(sum(파드의 현재 cpu 사용률) / targetAverageUtilization)
- 일정 시간마다 스케일 아웃 
  - avg(파드의 현재 cpu 사용률) / targetAverageUtilization > 1.1
- 일정 시간마다 스케일 인
  - avg(파드의 현재 cpu 사용률) / targetAverageUtilization < 0.9

쿠버네티스 1.18 부터는 `spec.behavior` 속성을 이용해 오토스케일링 동작을 상세하게 정의할 수 있음

> CPU 이외에도 Custom Metrics를 사용하여 임의의 지표에 따라 오토 스케일링을 할 수 있다


```shell
# 테스트를 위해서 쿠버네티스에 메트릭 서버를 설치해줘야 한다
minikube addons enable metrics-server

kubectl apply -f sample-hpa.yaml
kubectl apply -f sample-hpa-deployment.yaml

# 테스트를 위해 포트 포워딩
kubectl port-forward service/lb 8080:8080

# hpa 상태 감시
kubectl get hpa sample-hpa --watch

# apache bench 를 이용하여 부하 생성
ab -n 100000 -c 200 http://localhost:8080/

# cpu 메트릭 수집이 안되는 현상 발견 (오토 스케일 동작 x)
kubectl top po -A
kubectl get --raw /apis/metrics.k8s.io/v1beta1/namespaces/default/pods | jq ''

# 현재 사용중인 minikube 버전에서 알려진 버그로 보임
# ref link: https://github.com/kubernetes/minikube/issues/13898
minikube stop 

# 임시해결
minikube start --extra-config=kubelet.housekeeping-interval=10s

# cpu 메트릭 수집 확인 (10s interval)
kubectl top po -A

# apache bench 를 이용하여 부하 생성
# sample-hpa   Deployment/sample-hpa-deployment   0%/50%    1         10        1          30m
# sample-hpa   Deployment/sample-hpa-deployment   0%/50%    1         10        1          31m
# sample-hpa   Deployment/sample-hpa-deployment   100%/50%   1         10        2          31m
# sample-hpa   Deployment/sample-hpa-deployment   50%/50%    1         10        2          32m
# sample-hpa   Deployment/sample-hpa-deployment   52%/50%    1         10        2          32m
# sample-hpa   Deployment/sample-hpa-deployment   50%/50%    1         10        2          32m
```

## VerticalPodAutoscaler

파드에 할당하는 cpu/memory 리소스를 스케일링
