## Jaeger 설치하기
jaeger를 통해 각 서비스 구간 구간 별로 모니터링 가능

**addon 전체 설치하기 (추천)**
```bash
cd {istio-dir}
kubectl apply -f samples/addons
``` 
 
**jaeger만 설치하기**
``` bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.8/samples/addons/jaeger.yaml
```

## Jaeger Dashboard 보기 
istioctl dashboard jaeger

위 명령어를 실행하면 jaeger 대시보드를 로컬에서 접속할 수 있다



## Jaeger Dashboard 외부로 노출하기

인그레스를 통해 jaeger dashboard를 외부로 노출
```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: jeager
  namespace: istio-system
spec:
  rules:
    - host: ""
      http:
        paths:
          - path: /
            backend:
              serviceName: trace
              servicePort: 80
```

