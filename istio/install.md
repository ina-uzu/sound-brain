👍 참고 문서 : [https://istio.io/latest/docs/setup/install/operator/](https://istio.io/latest/docs/setup/install/operator/)

istio 설치 방법은 아래와 같은 것들이 있다. 

- istioctl
- istio operator
- helm
- multicluster(여러 클러스터에 걸쳐 서비스 메쉬를 구성할때)

## Install With Operator

istioctl로 수동으로 운영 컴포넌트를 배포하고 관리하지 말고, 전용 operator를 설치해서 관리하자~

버전 업데이트도 operator CR만 수정하면 알아서 됨

### 0. Install istioctl

파일 다운받고 binary를 /usr/local/bin으로 이동시켜 istioctl을 사용할 수 있도록 합니당

```bash
curl -sL https://istio.io/downloadIstioctl | sh -
cp istio-1.8.1/bin/istioctl /usr/local/bin
```

### 1. Deploy the Istio operator

```bash
istioctl operator init
```

이 명령어를 수행하면 `istio-operator` 네임스페이스가 생성되고, 여기에 아래의 resource들이 생성된다. 

- operator CRD
- operator controller deployment
- service to access operator metrics
- Istio operator RBAC rules

**Watch Namespaces**

기본적으로 operator는 **istio-system** 네임스페이스에만 운영 컴포넌트를 띄우고 watch하고 있다. 

다른 namespace에 운영 컴포넌트를 띄우고 싶다면  `--watchedNamespaces`를 사용하여 지정해준다.

```bash
# 여러 네임스페이스 지정 가능
istioctl operator init --watchedNamespaces=istio-namespace1,istio-namespace2
```

### 2. Install istio

적절한 configuration profile을 지정해서 istio를 설치하자.

`IstioOperator` 라는 커스텀 리소스를 생성하여 Operator를 띄우면, 이 Operator에 의해 운영 component가 생성된다. 

```bash
# istio 운영 component가 뜰 namespace 생성
kubectl create ns istio-system 

kubectl apply -f - <<EOF
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
  name: example-istiocontrolplane
spec:
  profile: demo 
EOF
```

+) **Configuration Profile** 

built-in profile들은 아래와 같다. x 표시 된 것들이 설치된다

IstioOperator를 생성하고 나면, istio-system 네임스페이스에 컴포넌트들이 배포된 걸 확인할 수 있다. 

(demo profile을 선택했기 때문에 istiod, istio-ingressgateway, istio-egressgateway가 다 생성되었다)


아래와 같이 IstioOperator spec 필드를 통해 좀더 세세하게 배포 설정을 할 수 있다. 

자세한 내용은  [IstioOperator Options 참고](https://istio.io/latest/docs/reference/config/istio.operator.v1alpha1/#IstioOperatorSpec)

```bash
$ kubectl apply -f - <<EOF
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
  name: example-istiocontrolplane
spec:
  profile: default
  components:
    pilot:
      k8s:
        resources:
          requests:
            memory: 3072Mi
    egressGateways:
    - name: istio-egressgateway
      enabled: true
EOF
```

## Uninstall

먼저 IstioOperator CR을 삭제하여 istio 운영 컴포넌트들을 삭제한다. 

```bash
kubectl delete istiooperators.install.istio.io -n istio-system example-istiocontrolplane
```

이제 istio operator를 삭제한다. 

```bash
# revision을 생략하면 모든 revision의 operator 삭제 
istioctl operator remove --revision <revision>
```

## Upgrade

TODO 

[https://istio.io/latest/docs/setup/install/operator/#in-place-upgrade](https://istio.io/latest/docs/setup/install/operator/#in-place-upgrade)
