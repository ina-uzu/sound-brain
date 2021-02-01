# sidecar-injection
### **Manual**

istioctl kube-inject 명령어를 통해 envoy가 같이 띄워진 yaml 파일을 얻을 수 있다!

```bash
istioctl kube-inject -f sample/app.yaml | kubectl apply -f -
```

### **Auto sidecar injection**

```bash
kubectl label namespace <namespace> istio-injection=enabled
```

앱을 띄울 네임스페이스에  `istio-injection=enabled` 라벨을 걸면, 이후 배포되는 pod에 자동으로 proxy container가 삽입된다.

**🤔 How?** 

istio가 설치되면 `pod 생성 event가 발생하면 사이드카로 envoy proxy를 자동으로 추가해주는 컨트롤러`가 

mutating webhook admission controller로 등록된다. 

```bash
kubectl get mutatingwebhookconfiguration -o yaml

# ... 생략
    name: sidecar-injector.istio.io
    namespaceSelector:
      matchLabels:
        istio-injection: enabled
```

### 참고

그 외에도 pod annotation, sidecar injection configmap 등을 통해 설정이 가능하다

아래 링크 참고
[https://happycloud-lee.tistory.com/104](https://happycloud-lee.tistory.com/104)
