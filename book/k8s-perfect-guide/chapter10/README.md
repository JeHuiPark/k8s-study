# 헬스 체크

Kubernetes는 Pod의 container 가용성을 검사하는 수단을 제공하며, 가용성 검사는 각 Node의 kubelet agent가 probe라는 구현체를 사용하여 진행한다 

**Probe Types**

|      Probe       | 역할                                                                             | 실패 시 동작                   | 
|:----------------:|--------------------------------------------------------------------------------|---------------------------|
|  Startup Probe   | 컨테이너가 시작되었는지 상태를 확인 <br> (실행시간이 오래걸리거나 컨테이너 초기화 작업이 필요한 경우에 활용하기에 적합)          | 컨테이너 재시작                  |
| Readiness Probe  | 컨테이너가 요청에 응답할 준비가 되었는지를 확인 <br> (백엔드 failover와 같이 일시적으로 서비스 불가능한 경우를 판단하기에 적합) | 트래픽 차단<br/>(파드를 재기동하지 않음) |
|  Liveness Probe  | 컨테이너가 정상 동작 중인지 확인 <br> (컨테이너 disk가 모두 고갈되었거나, 복구 불가능한 상태일 경우를 판단하기에 적합)       | 컨테이너 재시작                  |


> - Startup Probe는 컨테이너가 시작될 때 사용되며 한 번이라도 통과하면 컨테이너가 재시작 되기전까지 실행되지 않는다  
> - Readiness Probe와 Liveness Probe는 컨테이너가 Running 상태일 때 상태검사 루프에 의해 지속적으로 실행되며 각각은 독립적이다. (호출 순서 보장X)

일반적으로 type: LoadBalancer의 서비스로 로드 밸런서를 생성한 경우, 로드 밸런서에서 쿠버네티스 노드의 헬스 체크는 ICMP를 통한 단순한 체크만 하게 된다.  
따라서 파드 상태를 체크하여 트래픽을 제어하려면 Readiness Probe 또는 Liveness Probe를 제대로 설정해야 한다

## 헬스 체크 방식

| 헬스 체크 방식  | 내용                                             |
|-----------|------------------------------------------------|
| exec      | 명령어를 실행하고 종료 코드가 0이 아니면 실패                     |
| httpGet   | http get 요청을 실행하고 status code가 200~399가 아니면 실패 |
| tcpSocket | TCP 세션이 연결되지 않으면 실패                            |

### exec

```yaml
livenessProbe:
  exec:
    command: ["test", "-e", "/ok.txt"]
```

### httpGet

```yaml
livenessProbe:
  httpGet:
    path: /helath
    port: 80
    scheme: HTTP
    host: example.com
    httpHeaders: 
      - name: Authorization
        value: Bearer token
```

### tcpSocket
```yaml
liveness:
  tcpSocket:
    port: 80
```

### gRPC HealthCheck
gRPC에는 gRPC [헬스 체킹 프로토콜](https://github.com/grpc/grpc/blob/master/doc/health-checking.md) 을 사용한 헬스 체크 기능이 있다.  
[grpc_health_probe](https://github.com/grpc-ecosystem/grpc-health-probe) 를 사용하면 엔드포인트에 대해 헬스 체크를 할 수 있다. `exec.command` 에서 실행하기 위해 사전에 컨테이너 이미지에 grpc_health_probe 바이너리를 추가해 두어야 함
```yaml
livenssProbe:
  exec:
    command: ["grpc binary path", "-addr=:5000"]
```

## 헬스체크 간격

| 설정 항목               | 내용                                         |
|---------------------|--------------------------------------------|
| initialDelaySeconds | 첫 번째 헬스 체크 시작까지의 지연(failureThresold 만큼 연장) |
| periodSeconds       | 헬스 체크 간격                                   |
| timeoutSeconds      | 타임아웃                                       |
| successThreshold    | 성공이라고 판단하기까지의 체크 횟수                        |
| failureThreshold    | 실패라고 판단하기까지의 체크 횟수                         |

```yaml
livenessProbe:
  initialDelaySeconds: 5
  periodSeconds: 5
  timeoutSeconds: 1
  successThreshold: 1
  failureThreshold: 1 
```

```shell
kubectl apply -f sample-healthcheck.yaml

kubectl describe pod sample-healthcheck | grep -E "Liveness|Readiness"

kubectl apply -f sample-liveness.yaml
kubectl exec -it sample-liveness -- rm -f /usr/share/nginx/html/index.html

# RESTARTS 카운트 증가 확인
kubectl get po sample-liveness --watch
# NAME              READY   STATUS    RESTARTS   AGE
# sample-liveness   1/1     Running   0          5s
# sample-liveness   1/1     Running   1 (0s ago)   9s

kubectl apply -f sample-readiness.yaml

# Readiness 헬스체크 실패 유도
kubectl exec -it sample-readiness -- rm -f /usr/share/nginx/html/50x.html

# Readiness 헬스체크 성공 유도
kubectl exec -it sample-readiness -- touch /usr/share/nginx/html/50x.html

# pod 상태 확인
kubectl get po sample-readiness --watch
# NAME               READY   STATUS    RESTARTS   AGE
# sample-readiness   1/1     Running   0          34s
# sample-readiness   0/1     Running   0          51s
# sample-readiness   1/1     Running   0          75s
# ReadinessProbe가 실패한 경우 Ready 상태가 아니므로 서비스에서 트래픽 전송을 차단한다, LivenessProbe와 다르게 컨테이너 재시작도 없다

# 헬스체크 이력정보
# kubectl describe po sample-readiness
# Events:
#   Type     Reason     Age                    From               Message
#   ----     ------     ----                   ----               -------
#   Normal   Scheduled  5m26s                  default-scheduler  Successfully assigned default/sample-readiness to minikube
#   Normal   Pulled     5m26s                  kubelet            Container image "nginx:1.16" already present on machine
#   Normal   Created    5m26s                  kubelet            Created container nginx-container
#   Normal   Started    5m26s                  kubelet            Started container nginx-container
#   Warning  Unhealthy  4m18s (x8 over 4m36s)  kubelet            Readiness probe failed: ls: cannot access '/usr/share/nginx/html/50x.html': No such file or directory
```

## 파드 Ready++(ReadinessGate)
특정한 경우에 시스템 전체 관점에서 파드가 정말 Ready 상태인지 추가로 체크하고 싶은 경우 '파드 ready++' 기능을 이용한다  

ReadinessGate를 통과할 때까지 서비스 전송 대상에 추가되지 않고 롤링 업데이트 시 다음 파드 기동으로 이동하지 않는다.

```shell
kubectl apply -f sample-readinessgate.yaml

# 서비스 전송대상에 아무것도 등록되지 않은 것을 확인할 수 있다
kubectl describe svc sample-readinessgate

kubectl get po sample-readinessgate -o wide
# NAME                   READY   STATUS    RESTARTS   AGE   IP            NODE       NOMINATED NODE   READINESS GATES
# sample-readinessgate   1/1     Running   0          24s   172.17.0.10   minikube   <none>           0/1

curl -X PATCH -H "Content-Type: application/json-patch+json" \
-d '[{"op": "add", "path": "/status/conditions/-", "value": {"type": "amsy.dev/sample-condition", "status": "True"}}]' \
http://localhost:8001/api/v1/namespaces/default/pods/sample-readinessgate/status

kubectl get po sample-readinessgate -o wide
# NAME                   READY   STATUS    RESTARTS   AGE   IP            NODE       NOMINATED NODE   READINESS GATES
# sample-readinessgate   1/1     Running   0          62s   172.17.0.10   minikube   <none>           1/1

# 서비스 전송대상에 파드가 등록된 것을 확인할 수 있다
kubectl describe svc sample-readinessgate
```

## Startup Probe를 사용한 지연체크

```shell
kubectl apply -f sample-startup.yaml

# Startup Probe 헬스체크 성공 유도
kubectl exec -it sample-startup -- touch /root/startup

# 헬스체크 로그 확인
kubectl exec -it sample-startup -- head /root/log
# [Sun Jun 26 06:32:32 UTC 2022] startup
# [Sun Jun 26 06:32:35 UTC 2022] startup
# [Sun Jun 26 06:32:38 UTC 2022] startup
# [Sun Jun 26 06:32:41 UTC 2022] startup
# [Sun Jun 26 06:32:44 UTC 2022] startup
# [Sun Jun 26 06:32:47 UTC 2022] startup
# [Sun Jun 26 06:32:50 UTC 2022] startup
# [Sun Jun 26 06:32:50 UTC 2022] readiness
# [Sun Jun 26 06:32:53 UTC 2022] readiness
# [Sun Jun 26 06:32:53 UTC 2022] liveness
```

## 컨테이너 라이프사이클

**restartPolicy**

| restartPolicy | 내용                                 |
|---------------|------------------------------------|
| Always        | 파드가 정지하면 항상 파드를 재기동                |
| OnFailure     | 예상치 못하게 파드가 정지한 경우 (종료코드가 0이 아닌경우) |
| Never         | 재기동하지 않음                           |

## Init Container
파드 내부에서 메인이 되는 컨테이너를 기동하기 전에 별도의 컨테이너를 기동하는 기능

`spec.initContainers` 속성에 복수로 지정할 수 있으며, 위에서부터 하나씩 컨테이너가 기동된다

**사용사례**
- 저장소에서 파일
- 컨테이너 기동 지연
- 동적 설정파일 처리
- 서비스 생성 확인
- 등등 메인 컨테이너를 기동하기전 필요한 체크 작업

```shell
kubectl apply -f sample-initcontainer.yaml
kubectl get po sample-initcontainer --watch
# NAME                   READY   STATUS     RESTARTS   AGE
# sample-initcontainer   0/1     Init:0/2   0          0s
# sample-initcontainer   0/1     Init:0/2   0          1s
# sample-initcontainer   0/1     Init:1/2   0          22s
# sample-initcontainer   0/1     Init:1/2   0          23s
# sample-initcontainer   0/1     PodInitializing   0          33s
# sample-initcontainer   1/1     Running           0          34s
kubectl exec -it sample-initcontainer --cat /usr/share/nginx/html/index.html
# Defaulted container "nginx-container" out of: nginx-container, output-1 (init), output-2 (init)
# 1st
# 2nd
```

## postStart/preStop
기동 직후와 종료 직전에 임의의 명령어를 실행

```shell
kubectl apply -f sample-lifecycle-exec.yaml

kubectl exec -it sample-lifecycle-exec --  ls /tmp

kubectl delete -f sample-lifecycle-exec.yaml

kubectl exec -it sample-lifecycle-exec --  ls /tmp
```

postStart는 spec.containers[].command 실행과 거의 같은 타이밍에 실행된다 (비동기)
기동시에 필요한 파일을 배치하는 등의 초기화 작업은 초기화 컨테이너를 사용하거나 entrypoint 안에서 실행한 후에 기동하도록 해야한다

prestop은 종료 요청이 온 후 실행된다

**주의사항**
- 최소 한 번 실행된다 (여러 번 실행될 가능성도 있다)
- postStart에 타임아웃 설정을 할 수 없다
  - 실행중에는 probe도 실행되지 않는다
  - deadlock을 유발할 수 있는 프로그램은 실행하지 않도록 한다

## 파드의 안전한 정지와 타이밍

기동 중인 파드의 삭제 요청이 쿠버네티스 API 서버에 도창하면 비동기로 'preStop 처리 + SIGTERM 처리'와 '서비스에서 제외 설정'이 이루어진다.  
이 두가지 처리는 자율분산 시스템(Anonymous Decentralized System)이므로, 엄밀히 말하면 동시에 처리되는 것은 아니다.  
따라서 '서비스에서 제외 설정'이 끝나기 전에, 'preStop 처리 + SIGTERM 처리'에서 프로세스를 정지해 버리면 일부 요청에서 에러가 발생하기 때문에 주의해야한다

파드에는 `spec.terminationGracePeriodSecond`(기본값 30초)라는 설정값이 있으며, 이 시간 안에 'preStop 처리 + SIGTERM 처리'를 끝내야 한다.  
만일 이 시간 내에 처리가 끝나지 않는다면 SIGKILL 신호가 컨테이너에 전달되어 강제적으로 종료된다. preStop 설정이 없는 경우에는 그대로 SIGTERM 처리가 진행된다

> preStop 처리만으로 terminationGracePeriodSecond 시간을 전부 소용한 경우에는 2초만 SIGTERM 처리 시간으로 확보된다

kubectl 명령어로 파드를 삭제할 때 `--grace-preiod` 옵션을 사용하여 파드의 terminationGracePeriodSecond 를 변경하여 삭제할 수 있다

