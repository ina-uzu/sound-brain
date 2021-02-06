https://istio.io/latest/docs/ops/diagnostic-tools/proxy-cmd/

https://istio.io/latest/docs/ops/diagnostic-tools/istioctl-analyze/

## istioctl Analyze
```bash
# analyze live cluster
istioctl analyze --all-namespaces
 
 
# analyze local file
istioctl analyze samples/bookinfo/networking/bookinfo-gateway.yaml samples/bookinfo/networking/destination-rule-all.yaml
```

## envoy ↔ istiod(pilot) config 싱크 확인
```bash
# 전체 서비스 확인
istioctl proxy-status
 
 
# 특정 서비스 확인
istioctl proxy-status {확인하고 싶은 서비스의 pod}
```


## Verifying connectivity to Istiod
모든 사이드카 엔보이는 control plane인 istiod와 통신이 가능해야 한다. 
```bash
kubectl exec -it {확인하고 싶은 서비스의 pod} -- curl -sS istiod.istio-system:15014/debug/endpointz
```

## Debugging Envoy 
### envoy admin page (안추천)
기본적으로 각 사이트카 엔보이의 15000번 포트에 어드민 페이지가 연결되어 있다. 
```bash
kubectl port-forward {보고 싶은 서비스의 pod} 15000:15000
```

해당 페이지를 포트포워딩하여 어드민 페이지에서 필요한 정보를 직접 보자. 

보통은 config_dump를 봐서 전체 config를 확인했는데, 내용이 너무 많아서 실제로 별 도움은 안되는 느낌이다. -ㅅ-



### istioctl 이용
istioctl proxy-config 명령어를 통해 엔보이 listener, cluster, endpoints, routes 설정을 예쁘게 볼 수 있다. 

( ingress(egress) gateway에 연결된  listener, cluster(업스트림), endpoints, routes 조회 )

```bash
istioctl proxy-config listener -n istio-system istio-ingressgateway-57f8997d4-rmrtk
istioctl proxy-config cluster -n istio-system istio-ingressgateway-57f8997d4-rmrtk
istioctl proxy-config route -n istio-system istio-ingressgateway-57f8997d4-rmrtk
```

특히나 istiod에서 기본으로 밀어넣어주는 config에 대한 것만 조회할 수도 있다. 
```bash
istioctl proxy-config bootstrap -n istio-system istio-ingressgateway-57f8997d4-rmrtk
```
