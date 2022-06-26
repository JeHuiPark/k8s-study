# Manifest 범용화

## 헬름

skip

## Kustomize

쿠버네티스 커뮤니티 sig-cli가 제공하는 매이페스트 템플레이팅 툴이다.  
환경마다 매니페스트를 생성하거나 특정 필드를 덮어 쓰는 기능이 있어 효율적으로 메니페스트를 생성할 수 있다.

기본값을 덮어 쓰는 형태로 설정하는 것이 특징

### Hello World
```shell
kubectl kustomize resources-sample/
#apiVersion: v1
#kind: Service
#metadata:
#  name: sample-lb
#spec:
#  ports:
#  - nodePort: 30082
#    port: 8080
#    targetPort: 80
#  selector:
#    app: sample-app
#  type: LoadBalancer
#---
#apiVersion: apps/v1
#kind: Deployment
#metadata:
#  name: sample-deployment
#spec:
#  replicas: 3
#  selector:
#    matchLabels:
#      app: sample-app
#  template:
#    metadata:
#      labels:
#        app: sample-app
#    spec:
#      containers:
#      - image: nginx:1.16
#        name: nginx-container
```

### 네임스페이스 덮어 쓰기

쿠버네티스 특유의 구조를 의식하지 않아도 자동으로 바꿔준다

```shell
kubectl kustomize namespace-sample/
#apiVersion: v1
#kind: Service
#metadata:
#  name: sample-lb
#  namespace: sample-namespace
#spec:
#  ports:
#  - nodePort: 30082
#    port: 8080
#    targetPort: 80
#  selector:
#    app: sample-app
#  type: LoadBalancer
#---
#apiVersion: apps/v1
#kind: Deployment
#metadata:
#  name: sample-deployment
#  namespace: sample-namespace
#spec:
#  replicas: 3
#  selector:
#    matchLabels:
#      app: sample-app
#  template:
#    metadata:
#      labels:
#        app: sample-app
#    spec:
#      containers:
#      - image: nginx:1.16
#        name: nginx-container
```

### prefix/suffix 부여
```shell
kubectl kustomize prefix-sample
#apiVersion: v1
#kind: Service
#metadata:
#  name: prefix-sample-lb-suffix
#spec:
#  ports:
#  - nodePort: 30082
#    port: 8080
#    targetPort: 80
#  selector:
#    app: sample-app
#  type: LoadBalancer
#---
#apiVersion: apps/v1
#kind: Deployment
#metadata:
#  name: prefix-sample-deployment-suffix
#spec:
#  replicas: 3
#  selector:
#    matchLabels:
#      app: sample-app
#  template:
#    metadata:
#      labels:
#        app: sample-app
#    spec:
#      containers:
#      - image: nginx:1.16
#        name: nginx-container
```

### 공통 메타데이터(레이블/어노테이션) 부여

서비스의 셀렉터나 디플로이먼트의 파드 템플릿의 어노테이션 등도 자동으로 바뀐다  
> 특정 리소스에만 부여하고 싶은 경우 오버레이 방식을 사용한다

```shell
kubectl kustomize commonmeta-sample
#apiVersion: v1
#kind: Service
#metadata:
#  annotations:
#    annotations1: annotation1-val
#  labels:
#    label1: label1-val
#  name: sample-lb
#spec:
#  ports:
#  - nodePort: 30082
#    port: 8080
#    targetPort: 80
#  selector:
#    app: sample-app
#    label1: label1-val
#  type: LoadBalancer
#---
#apiVersion: apps/v1
#kind: Deployment
#metadata:
#  annotations:
#    annotations1: annotation1-val
#  labels:
#    label1: label1-val
#  name: sample-deployment
#spec:
#  replicas: 3
#  selector:
#    matchLabels:
#      app: sample-app
#      label1: label1-val
#  template:
#    metadata:
#      annotations:
#        annotations1: annotation1-val
#      labels:
#        app: sample-app
#        label1: label1-val
#    spec:
#      containers:
#      - image: nginx:1.16
#        name: nginx-container
```

### 이미지 덮어 쓰기

리소스에 사용되는 이미지를 바꿀 수 있다.  
> 특정 리소스만 업데이트하는 경우에는 오버레이 구조를 사용한다

```shell
kubectl kustomize image-sample
#apiVersion: v1
#kind: Service
#metadata:
#  name: sample-lb
#spec:
#  ports:
#  - nodePort: 30082
#    port: 8080
#    targetPort: 80
#  selector:
#    app: sample-app
#  type: LoadBalancer
#---
#apiVersion: apps/v1
#kind: Deployment
#metadata:
#  name: sample-deployment
#spec:
#  replicas: 3
#  selector:
#    matchLabels:
#      app: sample-app
#  template:
#    metadata:
#      labels:
#        app: sample-app
#    spec:
#      containers:
#      - image: amsy810/echo-nginx:v2.0
#        name: nginx-container
```

### 오버레이로 갚 덮어쓰기

오버레이 기능을 사용하여 세부적인 설정을 패치하여 변경할 수 있다 

```shell
kubectl kustomize overlay-sample/production
#apiVersion: v1
#kind: Service
#metadata:
#  name: sample-lb
#spec:
#  ports:
#  - nodePort: 30082
#    port: 8080
#    targetPort: 80
#  selector:
#    app: sample-app
#  type: LoadBalancer
#---
#apiVersion: apps/v1
#kind: Deployment
#metadata:
#  name: sample-deployment
#spec:
#  replicas: 100
#  selector:
#    matchLabels:
#      app: sample-app
#  template:
#    metadata:
#      labels:
#        app: sample-app
#    spec:
#      containers:
#      - image: nginx:production
#        name: nginx-container
```

이 기능을 사용하면 덮어 쓰기로 패치가 되기 때문에 디렉터리 구조는 다음과 같이 간단해진다.
```
.
├── resources
│   ├── kustomization.yaml
│   ├── manifest-1.yaml
│   └── manifest-2.yaml
├── dev
│   ├── kustomization.yaml
│   └── patch.yaml
├── stg
│   ├── kustomization.yaml
│   └── patch.yaml
└── prod
    ├── kustomization.yaml
    └── patch.yaml
```

### 컨피그맵/시크릿 동적 생성

생성된 컨피그맵 이름은 데이터 구조에서 계산된 해시값이 접미사로 부여된다

```shell
kubectl kustomize generate-sample
#apiVersion: v1
#data:
#  KE1: VAL1
#  sample.txt: |
#    this is test file.
#kind: ConfigMap
#metadata:
#  name: generated-configmap-82chh4b9t9
#---
#apiVersion: apps/v1
#kind: Deployment
#metadata:
#  name: sample-deployment
#spec:
#  replicas: 3
#  selector:
#    matchLabels:
#      app: sample-app
#  template:
#    metadata:
#      labels:
#        app: sample-app
#    spec:
#      containers:
#      - envFrom:
#        - configMapRef:
#            name: generated-configmap-82chh4b9t9
#        image: nginx:1.16
#        name: nginx-container
```

### kustomize 관련 kubectl 하위 명령

```shell
kubectl apply -k resources-sample
kubectl get -k  resources-sample
kubectl describe -k resources-sample
kubectl diff -k resources-sample
kubectl delete -k resources-sample
```