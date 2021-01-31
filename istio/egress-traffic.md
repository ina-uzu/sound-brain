# Egress Traffic

## 준비

istio에서 관리하는 pod들은 기본적으로 모든 outbound 트래픽이 envoy proxy를 통합니다. 

(inbound도 마찬가지지만)

그래서 클러스터 url에 대한 접근성은  proxy 구성에 따라 달라지게 되는데, default는 "모두 허용" 입니다.

특정 서비스만 허용, 특정 IP만 bypass 하는 등의 설정을 할 수 있습니다.

**먼저 예제 앱을 배포합시다**

```
# istio dir에서 
kubectl apply -f samples/sleep/sleep.yaml

# default namespace에 sidecar auto injection 옵션이 활성화 되지 않은 경우 
kubectl apply -f <(istioctl kube-inject -f samples/sleep/sleep.yaml
```

---

## 1) 외부 접근 옵션 확인 (outboundTrafficPolicy)

아래 명령어를 통해 outboundTrafficPolicy 모드를 확인할 수 있습니다. 

(istio operator를 통해 설치했다고 가정합니다)

아무런 output이 없는 경우 "ALLOW_ANY" 모드 입니다.

```
# istiooperator 확인
kubectl get istiooperator -n istio-system 

# outboundTrafficPolicy 확인
kubectl get istiooperator -n istio-system {istiooperatorName} -o jsonpath='{.spec.meshConfig.outboundTrafficPolicy.mode}'
```

ALLOW_ANY 모드에서 아래와 같이 외부 서비스로 요청을 보내보면, 

200으로 호출 가능한 상태인 것을 확인할 수 있습니다.

```
$ export SOURCE_POD=$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})

# 외부 서비스인 google에 요청 보내보기 
$ kubectl exec -it $SOURCE_POD -c sleep -- curl -I https://google.com

HTTP/2 200
date: Thu, 28 Jan 2021 15:51:40 GMT
content-type: text/html; charset=utf-8
content-length: 2016
vary: Accept-Encoding
vary: Origin
access-control-allow-origin: *
last-modified: Tue, 26 Jan 2021 04:57:21 GMT
strict-transport-security: max-age=15724800; includeSubDomains
```

---

## 2) outboundTrafficPolicy을 REGISTRY_ONLY로 변경

REGISTRY_ONLY 모드에서는 istio service registry에 등록된 서비스로 가는 요청이 아니면 허용되지 않습니다.

istiooperator CR을 아래와 같이 수정하여 outboundTrafficPolicy를 REGISTRY_ONLY로 변경해보겠습니다.

```
# istiooperator 확인
kubectl edit istiooperator -n istio-system {istiooperatorName}

```

kubectl edit 명령어로 istiooperator에 아래 부분을 추가해주세요

```
spec:
  meshConfig:
    outboundTrafficPolicy:
      mode: REGISTRY_ONLY
```

이제 다시 동일하게 sleep pod 안에서 google로 curl을 보내보면

이번에는 "command terminated with exit code 35" 에러가 나며 요청이 허용되지 않을 걸 확인할 수 있습니다.

---

## 3) REGISTRY_ONLY 모드에서 ServiceEntry 추가하기

REGISTRY_ONLY 모드에서 외부 서비스에 접근하기 위해서는 접근 host, 프로토콜별 [ServiceEntry](https://istio.io/latest/docs/concepts/traffic-management/#service-entries) 를 생성해주어야 합니다.

google 서비스를 ServiceEntry로 등록해보겠습니다.

```
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: google-ext
spec:
  hosts:
  - google.com
  ports:
  - number: 443
    name: https
    protocol: HTTPS
  resolution: DNS
  location: MESH_EXTERNAL
EOF
```

위와 같은 ServiceEntry 생성후에 다시  sleep pod 안에서 "https://google.com" 로 curl을 보내보면, 요청이 허용되어 정상적으로 200이 리턴되는 걸 볼 수 있습니다

(이때 ServiceEntry에 명시하지 않은 http로는 여전히 호출이 불가능합니다)
