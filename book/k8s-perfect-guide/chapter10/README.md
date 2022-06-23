# 헬스 체크

헬스 체크 방법 3 가지

| Probe 종류        | 역할                        | 실패 시 동작                   |
|-----------------|---------------------------|---------------------------|
| Liveness Probe  | 파드 내부의 컨테이너가 정상 동작 중인지 확인 | 컨테이너 재기동                  |
| Readiness Probe | 파드가 요청을 받아들일 수 있는지 확인     | 트래픽 차단<br/>(파드를 재기동하지 않음) |
| Startup Probe   | 파드의 첫 번째 기동이 완료되었는지 확인    | 다른 Probe 실행을 시작하지 않음      |


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