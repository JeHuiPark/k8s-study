# 보안

## 서비스 어카운트

쿠버네티스에는 사용자와 비슷한 개념으로 UserAccount와 ServiceAccount가 있다.

**사용자 어카운트**
- GKE에서는 구글 계정과 연결되거나, EKS에서는 IAM과 연결
- 쿠버네티스 관리 대상이 아님
- 클러스터 수준의 존재 
- 네임스페이스의 영향을 받지 않음

**서비스 어카운트**
- 쿠버네티스에서만 사용 됨
- 파드에서 실행되는 프로세스를 위해 할당
- 네임스페이스와 연결된 리소스
- 파드 기동 시 반드시 서비스 어카운트 한 개를 할당해야 함 (서비스 어카운트 기반 인증/인가를 하고 있음)
  - 미지정시 기본 서비스 어카운트 할당

```shell
# 서비스 어카운트 생성
kubectl create sa sample-serviceaccount

kubectl patch serviceaccount sample-serviceaccount \
-p '{"imagePullSecrets": [{"name": "myregistrykey"}]}'

# 서비스 어카운트를 생성할 때 지정하지 않은 시크릿 항목이 존재
kubectl get sa sample-serviceaccount -o yaml

kubectl apply -f sample-serviceaccount-pod.yaml

# 도커 레지스트리 인증정보 확인 
kubectl get po sample-serviceaccount-pod -o json | jq '.spec.imagePullSecrets' 
```

## RBAC (Role Based Access Control)
RBAC는 어떤 조작을 허용하는지를 결정하는 Role을 생성하고 서비스 어카운트 등의 사용자에게 롤을 연결하여 권한을 부여한다.  
AggregationRule을 사용하여 여러 롤을 집약한 롤을 생성할 수도 있다

- 클러스터 수준
  - 클러스터롤 & 클러스터롤 바인딩
- 네임스페이스 수준
  - 롤 & 롤 바인딩

**롤에 지정할 수 있는 실행 가능한 조작(verbs)**  

| 종류     | 개요      |
|--------|---------|
| *      | all     |
| create | 생성      |
| delete | 삭제      |
| get    | 조회      |
| list   | 목록 조회   |
| patch  | 일부 업데이트 |
| update | 업데이트    |
| watch  | 변경 감시   |

롤과 클러스터롤을 생성할 때 주의사항
- apiGroups
- deployment 리소스와 deployment/scale 서브 리소스는 개별적으로 지정

### 프리셋 클러스터롤

단순한 권한을 사용하는 경우에는 신규로 롤이나 클러스터롤을 생성하지 않고 프리셋을 사용할 수도 있다

| 클러스터롤 명      | 내용                         |
|---------------|----------------------------|
| cluster-admin | 모든 리소스 관리 가능               |
| admin         | 클러스터롤 편집 + 네임스페이스 수준의 RBAC |
| edit          | 읽기 쓰기                      |
| view          | 읽기 전용                      |

### 롤바인딩과 클러스터롤바인딩

- 하나의 롤 바인딩당 하나의 롤만 가능
- subjects에는 여러 사용자나 서비스 어카운트 지정 가능
- **롤바인딩**은 롤 또는 클러스터롤과 사용자를 연결할 수 있다
- **클러스터롤바인딩**은 클러스터롤과 사용자를 연결할 수 있다
- **롤바인딩**은 특정 네임스페이스에 롤 또는 클러스터롤에서 정의된 권한을 부여
- **클러스터롤바인딩**은 모든 네임스페이스에 클러스터롤에서 정의된 권한을 부여

