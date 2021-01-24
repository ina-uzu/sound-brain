# kubectl 
## basic
```bash
# context 변경
kubelctl config --kubeconfig=my-kube-config use-context {contextName}

# api resource & version
kubectl api-resources
kubectl api-versions

# resource version 등등 확인
kubectl explain replicaset 

# 죽은 pod log
kubectl logs nginx -p

# yaml 얻기 (dry-run)
kubectl run pod --image=nginx --restart=Never \ 
--requests=cpu=0.5,memory=128Mi \
--labels=foo=bar \
--serviceaccount=test-sa
--dry-run=client -o yaml > pod.yaml 

# rollout
kubectl rollout status deployment/myapp-deployment
kubectl rollout history deployment/myapp-deployment
kubectl rollout undo deployment/myapp-deployment

# rbac
kubectl create serviceaccount lister
kubectl create role lister-role --verb=get,watch,list --resource=pods
kubectl create rolebinding lister-rolebinding --role=lister-role --serviceaccount=default:lister

# override
k run test --image=busybox --command sleep 1000 --overrides='{"spec": {"nodeName": "node03"}}'
```

## json path / custom columns
- json path [https://kubernetes.io/docs/reference/kubectl/jsonpath/](https://kubernetes.io/docs/reference/kubectl/jsonpath/)
- custom columns [https://kubernetes.io/docs/reference/kubectl/cheatsheet/#formatting-output](https://kubernetes.io/docs/reference/kubectl/cheatsheet/#formatting-output)

```bash
kubectl get pods -o json
kubectl get pods -o=jsonpath='{@}'
kubectl get pods -o=jsonpath='{.items[0]}'
kubectl get pods -o=jsonpath='{.items[0].metadata.name}'
kubectl get pods -o=jsonpath="{.items[*]['metadata.name', 'status.capacity']}"
kubectl get pods -o=jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.startTime}{"\n"}{end}'
```

```bash
# List Services Sorted by Name
kubectl get services --sort-by=.metadata.name

kubectl get services --sort-by=.metadata.name --no-headers | tac
```

```bash
# custom columns
k get pv --sort-by=.spec.capacity.storage \
-o=custom-columns='NAME:.metadata.name,CAPACITY:.spec.capacity.storage'
```
