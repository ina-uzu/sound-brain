## Kiali 설치하기

**addon 전체 설치하기 (추천)**

```bash
cd {istio-dir}
kubectl apply -f samples/addons
```

**kiali만 설치하기**

```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.8/samples/addons/kiali.yaml
```

한번에 설치가 안될 수 있다. 

`no matches for kind "MonitoringDashboard" in version "monitoring.kiali.io/v1alpha1"`  요런 에러가 나온다면 위 명령어를 한 번 더 수행한다.

## Kiali Dashboard 보기

`istioctl dashboard kiali`

위 명령어를 실행하면 kiali 대시보드를 로컬에서 접속할 수 있다!

- 실제 트래픽을 바탕으로한 서비스 매쉬 토폴로지 확인
- request time, response, error 등등 확인 가능
- istio proxy config 관리 가능 (에러도 잡아준다)
- 앱 로그 확인 등등

대시보드에 대한 자세한 설명은 [여기를 참고하기!](https://istio.io/latest/docs/tasks/observability/kiali/)

## Kiali Dashboard 외부로 노출하기

istio-ingressgateway에 연결하여 kiali를 외부에서 접근하자. 

gateway, virtual 서비스를 생성해준다. 

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: addon-gateway
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway # use istio default ingress gateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "istio-bookinfo.dev.9rum.cc"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: addon-service
  namespace: istio-system
spec:
  hosts:
  - "istio-bookinfo.dev.9rum.cc"
  gateways:
  - addon-gateway
  http:
  - match:
    - uri:
        prefix: /kiali
    route:
    - destination:
        host: kialo
        port:
          number: 20001
```

**ingress를 사용하는 방법**

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kiali
  namespace: istio-system
spec:
  rules:
    - host: "host"
      http:
        paths:
          - path: /
            backend:
              serviceName: kiali
              servicePort: 20001
```
