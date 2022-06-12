kebectl이 쿠버네티스 마스터와 통신할 때는 접속 대상의 서버 정보, 인증 정보 등이 필요  
kubectl은 kubeconfig(~/.kube/config)에 저장된 정보를 사용

```shell
# 클러스터(prd-cluster) 정의를 추가, 변경
kubectl config set-cluster prd-cluster \
--server=https://localhost:6443

# 인증 정보 정의를 추가, 변경
kubectl config set-credentials admin-user \
--client-certificate= ./sample.crt \
--client-key=./sample.key \
--embed-certs=true

# 컨텍스트 정의(클러스터/인증 정보/네임스페이스 정의)를 추가, 변경
kubectl config set-context prd-admin \
--cluster=prd-cluster \
--user=admin-user \
--namepsace=default

# 컨텍스트 목록 표시
kubectl config get-contexts

# 컨텍스트 전환
kubectl config use-context prd-admin

# 현재 컨텍스트 표시
kubectl config current-context

# 명령어가 실행될 컨텍스트 지정
kubectl --context prd-admin get pod
```

또 다른 cli 도구 `kubectx`, `kubens`는 많은 클러스터나 네임스페이스를 관리해야 하는 경우에 유리함

```shell
# 컨텍스트 전환
kubectx docker-desktop

# 이전 컨텍스트로 전환
kubectx -

# 컨텍스트 삭제
kubectx -d prd-admin

# 네임스페이스 전환
kubens default
```

## 리소스 생성/수정/삭제

```shell
# 리소스 생성
kubectl create -f sample-pod.yaml

# 파드 목록 조회
kubectl get pods

# 리소스 삭제 (manifest 기반)
kubectl delete -f sample-pod.yaml

# 특정 리소스 삭제
kubectl delete pod sample-pod

# 특정 리소스 종류를 모두 삭제
kubectl delete pod --all
```

kubectl 명령어 실행은 바로 완료되지만, 쿠버네티스에서 실제 리소스 처리는 비동기로 실행된다.  
쿠버네티스 리소스 처리의 완료를 기다리려면 `--wait` 옵션을 사용

```shell
# 리소스 즉시 강제 삭제
kubectl delete -f sample-pod.yaml --grace-period 0 --force

# 리소스 업데이트 (리소스가 없을 경우엔 신규로 생성, 변경할 수 없는 속성도 있음)
# kubectl create가 아닌 kubectl apply를 권장 
kubectl apply -f sample-pod.yaml 
```

server-side apply를 이용하여 변경사항 충돌을 방지할 수 있음
```shell
kubectl delete -f sample-pod.yaml
kubectl apply -f sample-pod.yaml  --server-side
kubectl set image pod sample-pod nginx-container=nginx:1.17

# 변경사항 충돌이 감지되어 오류 발생
kubectl apply -f sample-pod.yaml --server-side
```

```shell
kubectl apply -f sample-pod.yaml --server-side --field-manager ci-tool

# ... sample-pod 리소스 변경

# conflict
kubectl apply -f sample-pod.yaml --server-side
```

## 파드 재기동
디플로이먼트 등의 리소스와 연결되어 있는 모든 파드를 재기동할 수 있다 (rollout restart)
> 리소스와 연결되어 있지 않은 단독 파드에는 사용할 수 없다
 
```shell
kubectl apply -f sample-deployment.yaml
kubectl apply -f sample-pod.yaml

kubectl rollout restart deployment sample-deployment

# error
kubectl rollout restart pod sample-pod
```

## generate resource name

kubectl apply명령어를 사용하는 것이 바람직하지만, kubectl create를 사용하여 난수가 있는 이름의 리소스를 생성할 수 있음
```shell
kubectl create -f sample-generatename.yaml
kubectl create -f sample-generatename.yaml
kubectl create -f sample-generatename.yaml
```

## 리소스 상태 체크와 대기

```shell
# sample-pod가 Ready 상태가 될 때까지 대기 (기본 타임아웃 30초)
kubectl wait --for=condition=Ready pod/sample-pod

# 모든 pod가 삭제될 때까지 대기
kubectl wait --for=delete pod --all --timeout=1s
```

## manifest 설계

```shell
# 여러 리소스가 정의된 manifest 적용
kubectl apply -f sample-multi-resource-manifest.yaml
kubectl delete -f sample-multi-resource-manifest.yaml

# 여러 manifest 동시 적용 (R 옵션은 Recursion)
kubectl apply -R -f sample-dir
kubectl delete -R -f sample-dir  
```

### 시스템 전체를 한 개의 디렉토리로 통합

규모가 크지 않을 경우에 적합하다
- whole-system
  - service-a.yaml (deployment + service)
  - service-b.yaml (deployment + service)
  - ...

### 시스템 전체를 특정 서브 시스템으로

내부 정책에 따라 부서별로 구분이 필요하거나 그룹핑이 필요할 경우에 사용되는 형태  
directory가 조직 구성에 따라 나누어졌다면 권한 설정에서 장점을 취할 수 있다

- whole-system
  - subsystem-a
    - service-a.yaml (deployment + service)
    - service-b.yaml (deployment + service)
    - ...
  - subsystem-b

### 마이크로서비스별로
가독성이 높아진다는 장점이 있지만, 적용 순서 제어가 어렵다는 단점이 있다

- whole-system
  - service-a
    - deployment.yaml
    - service.yaml
    - ...
  - service-b
    - deployment.yaml
    - service.yaml
    - ...
  - ...

## 어노테이션과 레이블

- `어노테이션`은 시스템 구성 요소가 사용하는 메타데이터
- `레이블`은 리소스 관리에 사용하는 메타데이터

어노테이션과 레이블은 '\[접두사\]/키:값'으로 구성된다. 접두사 부분은 옵션으로, 지정하는 경우 DNS 서브 도메인 형식이어야 한다. ([참고자료](https://kubernetes.io/ko/docs/concepts/overview/working-with-objects/annotations/#%EB%AC%B8%EB%B2%95%EA%B3%BC-%EC%BA%90%EB%A6%AD%ED%84%B0-%EC%85%8B))

```shell
kubectl apply -f sample-annotations.yaml

kubectl annotate pods sample-annotation annotation3=val3

# 어노테이션 오버라이드
kubectl annotate pods sample-annotation annotation3=val3-new

kubectl get pod sample-annotation -o json | jq .metadata.annotations

# 어노테이션 삭제
kubectl annotate pods sample-annotation annotation3-
```

```shell
kubectl apply -f sample-label.yaml
kubectl label pods sample-label label3=val3
kubectl label pods sample-label label3=val3-new --overwrite
kubectl get pod sample-label -o json | jq .metadata.labels
kubectl label pods sample-label label3-
kubectl get pod sample-label -o json | jq .metadata.labels
```

레이블은 크게 두 가지 용도로 사용된다
- 개발자가 사용하는 레이블
  - 많은 리소스를 효율적으로 관리하는 데 유용하게 사용된다
- 시스템이 사용하는 레이블

```shell
# label1=val1과 label2 레이블을 가진 파드를 조회
kubectl get pods -l label1=val1,label2

# label1, lable2 레이블을 표시
kubectl get pods -L label1,label2

# 모든 레이블 표시
kubectl get pods --show-labels
```

### 시스템이 사용하는 레이블
예시로, 파드 수를 유지하는 리소스(ReplicaSet)에서는 레이블에 부여된 파드 수를 계산하여 레플리카 수를 관리한다.
대상 파드 수가 레플리카 수의 설정보다 많은 경우에는 파드 중 하나를 삭제하고, 반대로 부족한 경우에는 새로운 파드를 생성한다.

**주의**: `app=A` 레이블을 가진 파드가 세 개가 되도록 레플리카 수를 지정하고 있다면,
실수로 `app=A` 레이블을 부여한 파드를 별도로 생성하게 되면 기존에 생성된 파드가 정지될 수도 있다.

### 권장 레이블

쿠버네티스에서는 다양한 에코시스템에서 공통화하기 위해 아래와 같이 레이블을 지정할 것을 권장하고 있다. ([참고자료](https://kubernetes.io/ko/docs/concepts/overview/working-with-objects/common-labels/)) 

| 레이블 키 이름                     | 개요                        |
|------------------------------|---------------------------|
| app.kubernetes.io/name       | 애플리케이션 이름                 |
| app.kubernetes.io/version    | 애플리케이션 버전                 |
| app.kubernetes.io/component  | 애플리케이션 내 구성 요소            |
| app.kubernetes.io/part-of    | 애플리케이션이 전체적으로 구성하는 시스템 이름 |
| app.kubernetes.io/instance   | 애플리케이션이나 시스템을 식별하는 인스턴스명  |
| app.kubernetes.io/managed-by | 이 애플리케이션을 관리하는 데 사용되는 도구  |

## prune

`--prune` 옵션은 manifest에서 삭제된 리소스를 감지하여 자동으로 삭제한다

```shell
kubectl apply -f prune-test-dir

# sample-1 manifest 삭제
mv prune-test-dir/sample-1.yaml prune-test-dir/sample-1.yaml.back 
kubectl apply -f prune-test-dir --prune -l system=a
kubectl get pods
```

이 기능은 **레이블과 일치하는 전체 리소스의 목록**에서 kubectl apply로 읽어들인 manifest 안에 포함되지 않는 리소스를 모두 삭제하는 구조로 동작한다.  
때문에 `--all` 옵션을 사용하면 클러스터 리소스를 기준으로 동작하기 때문에 조심해야 한다.  
마찬가지로 다른 디렉토리에 동일한 레이블을 가진 리소스가 있다면 그 리소스도 삭제될 수 있다

## diff

```shell
kubectl apply -f sample-pod.yaml 
kubectl set image pod sample-pod nginx-container=nginx:1.15
kubectl diff -f sample-pod.yaml

# 상태코드 확인
echo $?
```
`kubectl diff` 명령어의 종료 코드는 실제 클러스터에 등록된 정보와 manifest 간에 차이가 있으면 1을 반환하고, 차이가 없으면 0을 반환한다

## api-resources

사용 가능한 리소스 목록 조회

```shell
kubectl api-resources

# 네임스페이스에 속하는 리소스
kubectl api-resources --namespaced=true

# 네임스페이스에 속하지 않는 리소스
kubectl api-resources --namespaced=true
```

쿠버네티스에서 취급하는 리소스는 네임스페이스 수준의 리소스와 클러스터 수준의 리소스, 두 종류가 있다.

## 리소스 조회

```shell
kubectl get pods
kubectl get pods sample-pod
kubectl get pods -l label=val1 --show-labels

# --output (json/yaml/custom-columns/jsonpath/go-template)
kubectl get pods -o wide
kubectl get pods -o yaml
kubectl get pods -o yaml sample-pod

# Custom Columns
kubectl get pods -o custom-columns="NAME:{.metadata.name}, NodeIP:{.status.hostIP}, containers:{.spec.containers[].name}"

# json path
kubectl get pods sample-pod -o jsonpath="{.metadata.name}"

# 배열 필터링
kubectl get po sample-pod -o jsonpath="{.spec.containers[?(@.name == 'nginx-container')].image}"

# go template
kubectl get pods -o go-template="{{range.items}}{{.metadata.name}}:{{range .spec.containers}}{{.image}}{{end}}{{end}}"

# 노드 목록 조회
kubectl get nodes

# 거의 모든 종류의 리소스 조회
kubectl get all

# 모든 리소스 조회
kubectl get $(kubectl api-resources --namespaced=true --verbs=list -o name | tr '\n' ',' | sed -e 's|,$||g')

# 리소스 상태 감시
kubectl get pods --watch
kubectl get pods --watch --output-watch-events
```

## 리소스 상세 정보 조회

```shell
kubectl describe pod sample-pod
kubectl describe node minikube
```

## 컨테이너에서 명령어 실행

```shell
# 파드 내부의 컨테이너에서 /bin/ls 실행
kubectl exec -it sample-pod -- /bin/ls

# 여러 컨테이너에 존재하는 파드의 특정 컨테이너에서 /bin/ls 실행
kubectl exec -it sample-pod -c nginx-container -- /bin/ls

# 파드 내부의 컨테이너에서 /bin/bash 실행
kubectl exec -it sample-pod -- /bin/bash

# bin/bash에 인수 전달
kubectl exec -it sample-pod -- /bin/bash -c "ls --all --classify | grep lib"
```

## 포트 포워딩

```shell
kubectl apply -f sample-pod.yaml
kubectl port-forward sample-pod 8888:80
curl -I localhost:8888

kubectl apply -f sample-deployment.yaml
kubectl port-forward deployment/sample-deployment 8888:80
```

## 로그 출력

```shell
kubectl logs sample-pod

kubectl logs sample-pod -c nginx-container

kubectl logs -f sample-pod

kubectl logs --since=1h --tail=10 --timestamps=true sample-pod

kubectl logs -f --selector app=nginx

# 오픈 소스 Stern을 사용하면 로그를 더욱 편리하게 출력할 수 있음
brew install stern

stern sample-deployment-
```

## 컨테이너 - 로컬 머신 파일복사

```shell
kubectl cp sample-pod:etc/hostname ./hostname
cat hostname

kubectl cp hostname sample-pod:/tmp/newfile
kubectl exec -it sample-pod -- ls /tmp
```
## kubectl 플러그인 관리자 krew
kubectl은 하위 명령어를 확장할 수 있도록 플러그인을 지원한다  
[krew install guide](https://krew.sigs.k8s.io/docs/user-guide/setup/install/)

```shell
# 플러그인 목록 확인
kubectl plugin list

kubectl krew install tree rolesum sort-manifests open-svc view-serviceaccount-kubeconfig
```

## kubectl 디버깅

```shell
kubectl -v=6 get pod

kubectl -v=8 apply -f sample-pod.yaml
```

## kubectl etc tip

kube-ps1은 bash나 zsh의 프롬프트에 현재 작업 중인 쿠버네티스 클러스터와 네임스페이스를 표시한다

```shell
brew install kube-ps1
# install guide : https://github.com/jonmosco/kube-ps1
kubeon
kubeoff -g # 기본설정을 off로 변경

# 컨테이너 환경이나 애플리케이션에 문제가 있어서 디버깅이 필요할 경우
# 일시적으로 nginx 이미지로 웹 서버가 아닌 셸을 기동 
kubectl run --image=nginx:1.16 --restart=Never --rm -it sample-debug --comand -- /bin/sh
```