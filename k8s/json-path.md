## json path

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

kubectl get services --sort-by=.metadata.name --no-headers=true | tac
```

```bash
# custom columns
k get pv --sort-by=.spec.capacity.storage \
-o=custom-columns='NAME:.metadata.name,CAPACITY:.spec.capacity.storage'
```
