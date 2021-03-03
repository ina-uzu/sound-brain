## Troubleshooting
[Kubernetes+-CKA-+1000+-+Troubleshooting.pdf](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/95aeb15c-f635-4550-a142-07633823df90/Kubernetes-CKA-1000-Troubleshooting.pdf)

```
command
arg
image
selector / label
volume
conf file name
```

- 일단 모든 pod, svc, ep ... 확인해서 전체적으로 문제있는 게 뭔지 본다

```bash
kubectl get pod -A
kubectl get svc -A
kubectl get ep -A
```

---

## App

### service

- env 등으로 설정된 service name이 맞나 확인
- service port & target port 확인
- service selector 확인
- service nodePort 확인

### Deploy / Pod

- env 확인
- 안뜬 pod이 있나 확인
- pending 상태이면 → 노드 상태 & scheduler 떠있나 확인

## Control Plane

```bash
cd /etc/systemd/system/kubelet.service.d
cat {kubeadmConfigFileName}.conf

# kubelet config file path 확인 
grep -i staticPod /var/lib/kubelet/config.yaml
```

- static pod 수정
- command
- volume
- conf file name
- image name

## Worker Node

- kubelet이 돌고 있나 확인

```bash
journalctl -u kubelet

# 안 돌고 있으면
systemctl kubelet start
 
# 로그 확인
journalctl -u kubelet -f
```

- api server ip & port 확인

```bash
ssh node01
cat /etc/kubernetes/kubelet.conf # 변경
systemctl kubelet restart # 재시작해야 적용된다
```

- pod = kube-proxy, coredns, flannel

## Network

- pod network add on 설치 되었나 확인 ( k get pod -A해서 관련 pod 떠있나 확인)

```bash
# add on 설치
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

- connection to 30081이 나오면
- kube dns 확인 (포트 연결도 안 된다)

```bash
The kube-dns service is not working as expected. 

The first thing to check is if the service has a valid endpoint? 

Does it point to the kube-dbs/core-dns ?

Run: kubectl -n kube-system get ep kube-dns

If there are no endpoints for the service, inspect the service and make sure it uses the correct selectors and ports.

Run: kubectl -n kube-system descrive svc kube-dns

Note that the selector used is: k8s-app=core-dns

If you compare this with the label set on the coredns deployment and its pods, you will see that the select should be k8s-app=kube-dns

Modify the kube-dns service and update the selector to k8s-app=kube-dns
```
