# 컨피그 & 스토리지
## 영구 볼륨

영구 볼륨은 기본적으로 네트워크를 통해 디스크를 어태치 하는 디스크 타입이다

영구 볼륨을 생성할 때 아래와 같은 설정이 가능하다
- 레이블
- 용량
- [접근 모드](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#volume-mode)
- [Reclaim Policy](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#reclaim-policy) 
- 마운트 옵션
- 스토리지 클래스
- 각 플러그인 특유의 설정

동적 프로비저닝이 불가능한 환경에서는 용구 볼륨을 미리 준비해 두어야 한다

- 영구 볼륨 클레임은 영구 볼륨 리소스를 요청한다
- 영구 볼륨 클레임에서 요청한 용량과 영구 볼륨 리소스의 용량은 같지 않다
  - 동적 프로비저닝이 아니라면 영구 볼륨 클레임보다 큰 영구 볼륨 리소스가 할당될 수 있다

### 동적 프로비저닝

영구 볼륨 클레임이 생성되는 타이밍에 동적으로 영구 볼륨을 생성하고 할당한다 (용량 낭비가 발생하지 않는다)
> 어떤 영구 볼륨을 생성할 지 정의한 스토리지 클래스를 생성해야 한다.

```shell
kubectl apply -f sample-storageclass.yaml
kubectl get storageclass

kubectl apply -f sample-pvc-dynamic.yaml
kubectl get pvc
# pvc- 접두사가 붙은 볼륨이 생성 됨
kubectl get pv

# 볼륨 프로비저너를 사용하는 파드 생성 
kubectl apply -f sample-pvc-dynamic-pod.yaml
kubectl describe pvc sample-pvc-dynamic | grep -i used
```

**영구 볼륨 할당 타이밍 제어**  
스토리지클래스의 `volumeBindingMode` 속성을 이용하여 영구 볼륨이 할당되는 타이밍을 제어할 수 있다

| 설정값                  | 개요                     |
|----------------------|------------------------|
| Immediate (default)  | 즉시 영구 볼륨이 생성 됨         |
| WaitForFirstConsumer | 처음으로 사용될 때 영구 볼륨이 생성 됨 |

### 영구 볼륨을 블록 장치로 사용

영구 볼륨은 파일 시스템으로 어태치하지 않고 블록 장치로 어태치할 수도 있다
> minikube 는 block volume 프로비저닝을 지원하지 않음   

```shell
kubectl apply -f sample-pvc-block.yaml
kubectl apply -f sample-pvc-block-pod.yaml
```

### 영구 볼륨 클레임 확장 (리사이즈)

열구 볼륨 클레임 조정은 디스크 크기를 확장할 수 있지만 축소할 수는 없다
> minikube 환경(k8s.io/minikube-hostpath 클래스)은 지원하지 않음 ([참고자료](https://kubernetes.io/ko/docs/concepts/storage/storage-classes/#%EB%B3%BC%EB%A5%A8-%ED%99%95%EC%9E%A5-%ED%97%88%EC%9A%A9))

### volumeMount 몇 가지 옵션

- readOnly
- subPath (특정 디렉터리를 루트로 마운트)

