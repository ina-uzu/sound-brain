# Ingress Traffic

istio에서 ingress 리소스가 아니라 **istio gateway** 라는 걸 통해 서비스를 외부로 노출시킨다.

Gateway, VirtualService, DestinationRule 이라는 CRD를 통해 세부 설정을 할 수 있다.

**Gateway**

- HTTP/TCP 프로토콜 지정
- TLS 설정

**VirtaulService**

- routing rule 적용
- 쿠키, 헤더, uri 등 활용할 수 있음

**DestinationRule**

- subset 정의 (label 통해)
- loadbalancer 등 서비스로 트래픽 정책 결정

---

### Gateway

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: bookinfo-gateway
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - $MY_INGRESS_GATEWAY_HOST
```

위와 같은 crd를 생성하면, istio-system 네임스페이스에 loadbalancer 타입의 istio-ingressgateway service가 생성된다.

이를 NodePort 타입으로 변경해도 된다. 

```yaml
# nodeport 
echo $(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
```

## Virtual Service

Gateway에서 넘어온 트래픽을 받아주는 애이다. DestinationRule과 함께 라우팅 룰을 설정할 수 있다

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo
spec:
  hosts:
  - $MY_INGRESS_GATEWAY_HOST
  gateways:
  - bookinfo-gateway.default.svc.cluster.local

  # routing rule을 정의한다
  http:
  - match:
    - uri:
        exact: /productpage
    - uri:
        exact: /login
    - uri:
        exact: /logout
    - uri:
        prefix: /static
    route:
    - destination:
        host: productpage # 위의 url로 요청이 들어오면 productpage:9080으로 
        port:
          number: 9080
```

- hosts
    - *service* 이름
    - service registry 또는 ServiceEntry(external Mesh) 에서 lookup
    - short 네임은 대신 full 네임 권장 (`reviews` interpreted `reviews.default.svc.cluster.local`)
    - wildcard(*) 불가능 하지만 `.default.svc.cluster.local`, `.svc.cluster.local` 가능
    - 라우팅할 destination 이 없으면 무시(drop)됨
- spec.[http/tls/tcp].match
    - 라우팅 조건을 정의합니다.
    - exact, prefix, regex
    - 정의 가능 조건 : uri, method, headers, port, source Labels, gateways, queryParams
- spec.[http/tls/tcp].route
    - default 트래픽 라우팅 룰셋을 정의
- spec.[http/tls/tcp]…route.destination
    - (필수) 요청/연결을 포워딩해야 하는 *service* 인스턴스 고유한 식별자
- spec.[http/tls/tcp].*.destination.host
    - 타켓 *service* 인스턴스
- spec.[http/tls/tcp].*.destination.subset
    - **DestinationRule**의 subset
- spec.gateways
    - 라우트 해야하는 gateway, sidecar 이름
- spec.exportTo
    - 현재 virtual service 를 export 하는 네임스페이스를 정의

## DestinationRule

라우팅 발생 후 트래픽을 어떻게 처리할지 정의한다.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: productpage
spec:
  host: productpage
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
  subsets:
  - name: v1
    labels:
      version: v1
```

- subset 정의 (label 통해)
- 트래픽 정책
    - load balancing
    - connection pool

---

### reference

- [http://itnp.kr/post/istio-task-traffic-management-on-minikube](http://itnp.kr/post/istio-task-traffic-management-on-minikube)
