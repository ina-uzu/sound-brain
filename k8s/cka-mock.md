## kubectl
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

k run test --image=busybox --command sleep 1000 --overrides='{"spec": {"nodeName": "node03"}}'
```
---

### **k8s doc bookmark**

- kubectl cheatsheet:  [https://kubernetes.io/docs/reference/kubectl/cheatsheet/](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- installing kubeadm:  [https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm)
    - installing weave: [https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/#stacked-control-plane-and-etcd-nodes](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/#stacked-control-plane-and-etcd-nodes)
- k8s upgrade: [https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/#upgrading-control-plane-nodes](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/#upgrading-control-plane-nodes)
- etcd backup: [https://discuss.kubernetes.io/t/etcd-backup-and-restore-management/11019/12](https://discuss.kubernetes.io/t/etcd-backup-and-restore-management/11019/12)
- curl test pod: [https://kubernetes.io/docs/concepts/services-networking/connect-applications-service/#dns](https://kubernetes.io/docs/concepts/services-networking/connect-applications-service/#dns)
- kube-scheduler-flags: [https://kubernetes.io/docs/reference/command-line-tools-reference/kube-scheduler/](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-scheduler/)

[https://gist.github.com/jonatasbaldin/4e76846ce8fb17e5d2fa64bdea594930](https://gist.github.com/jonatasbaldin/4e76846ce8fb17e5d2fa64bdea594930)

→ 요렇게 누가 북마크 정리해둔 것도 있는데... 넘무 많아서 찾는 게 구찮다. 그냥 docs에서 검색하는 게 더 빠름

---

## Core Concepts

**Pod**

- If pod crashed and restarted, get logs about the previous instance

```bash
kubectl logs nginx -p
```

- Create a busybox pod that echoes 'hello world' and then exits

```bash
kubectl run busybox --image=busybox --command echo "hello world" --restart=Never
```

- Do the same, but have the pod deleted automatically when it's completed

```bash
kubectl run busybox --image=busybox -it --rm --restart=Never -- /bin/sh -c 'echo hello world'
```

- Create an nginx pod and set an env value as 'var1=val1'. Check the env value existence within the pod

```bash
kubectl run nginx --image=nginx --restart=Never --env=var1=val1

kubectl exec -it nginx -- env
```

**Deploy**

rollout history에 남도록 —recore=true

```bash
kubectl set image deployment.v1.apps/nginx-deploy nginx=nginx:1.17 --record=true

kubectl rollout history deploy nginx-deploy
#deployment.apps/nginx-deploy
#REVISION  CHANGE-CAUSE
#1         <none>
#2         kubectl set image deployment.v1.apps/nginx-deploy nginx=nginx:1.17 --record=true
```

**ReplicaSets**

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: frontend
  labels:
    app: guestbook
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: php-redis
        image: gcr.io/google_samples/gb-frontend:v3
```

## Scheduling

1. manual scheduling

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  nodeName: controlplane # here -> 직접 노드 지정하면 taint 무시됨
  containers:
  -  image: nginx
     name: nginx
```

2. taint 

```bash
kubectl taint node controlplane node-role.kubernetes.io/master:NoSchedule

# remove
kubectl taint node controlplane node-role.kubernetes.io/master:NoSchedule-
```


### static pod

- How many static pods exist in this cluster in all namespaces?

    → k get pod -A | grep controleplane

- ❤️ **NOTE!** staticPodPath 확인

```bash
systemctl status kubelt 

# or
ps -aux | grep kubelet

# --config=/var/lib/kubelet/config.yaml 확인

cat /var/lib/kubelet/config.yaml | grep staticPodPath
```

### **multiple scheduler **

flags : [https://kubernetes.io/docs/reference/command-line-tools-reference/kube-scheduler/](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-scheduler/)

- —scheduler-name=my-scheduler
- —leader-elect=false
- port 설정을 겹치지 않게 한다

```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    scheduler.alpha.kubernetes.io/critical-pod: ""
  creationTimestamp: null
  labels:
    component: my-scheduler
    tier: control-plane
  name: my-scheduler
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-scheduler
    - --address=127.0.0.1
    - --kubeconfig=/etc/kubernetes/scheduler.conf
    - --leader-elect=false # false
    - --port=0 # http는 안 쓰는 걸로 명시 
    - --scheduler-name=my-scheduler # 이름
    - --secure-port=10258 # 새 scheduler port 값
    image: k8s.gcr.io/kube-scheduler-amd64:v1.16.0
    imagePullPolicy: IfNotPresent
    livenessProbe:
      failureThreshold: 8
      httpGet:
        host: 127.0.0.1
        path: /healthz
        port: 10258 # 수정
        scheme: HTTPS # https -> secure port
      initialDelaySeconds: 15
      timeoutSeconds: 15
		# ...
```

Pod에 scheduler 명시 

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: annotation-second-scheduler
  labels:
    name: multischeduler-example
spec:
  schedulerName: my-scheduler # here
  containers:
  - name: pod-with-second-annotation-container
    image: k8s.gcr.io/pause:2.0
```

---

## Monitoring & Logging

- metric server 설치
- kubectl top node / kubectl top pod

---

## Lifecycle

### Secret

- 전체 env var를 secret 파일  전체로

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: k8s.gcr.io/busybox
      command: [ "/bin/sh", "-c", "env" ]
      envFrom:
      - secretRef:
          name: special-config # indent 주의
  restartPolicy: Never
```

### multi container

- volumeMounts & volumes

```yaml
spec:
  containers:
  - image : kodekloud/filebeat-configured
    name: sidecar
    volumeMounts:
      - mountPath: /var/log/event-simulator/
        name: log-volume
  - image: kodekloud/event-simulator
    imagePullPolicy: Always
    name: app
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /log
      name: log-volume
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: default-token-wn78b
      readOnly: true
# ...
  volumes:
  - hostPath:
      path: /var/log/webapp
      type: DirectoryOrCreate
    name: log-volume
```

---

## Cluster Maintenace

### Upgrade 

- [https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/#upgrading-control-plane-nodes](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/#upgrading-control-plane-nodes)

( 00는 업그레이드 해라 00는 하지마라 등..)

```bash
# latest stable version
kubeadm upgrade plan
```

1. master node
    1. drain node
    2. kubeadm upgrade
    3. kubectl & kubelet upgrade → restart
    4. uncordon node
2. worker node
    1. drain node
    2. kubeadm upgrade
    3. kubectl & kubelet upgrade → restart
    4. uncordon node

```bash
kubectl drain node01

# ssh node01
# upgrade kubeadm
apt update && \ apt install kubeadm=1.19.0-00
kubeadm upgrade node

# upgrade kubelet & kubectl
apt update && \ apt install kubelet=1.19.0-00 kubectl=1.19.00

sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

### Backup 

```bash
snapshot 생성 

-> kube-apiserver stop 

-> restore etcd (복원 snapshot 가지고 새로운 dir 지정해서) 

-> etcd pod 수정 (--data-dir = 새 dir & volume mount path) 
```

[https://github.com/mmumshad/kubernetes-the-hard-way/blob/master/practice-questions-answers/cluster-maintenance/backup-etcd/etcd-backup-and-restore.md](https://github.com/mmumshad/kubernetes-the-hard-way/blob/master/practice-questions-answers/cluster-maintenance/backup-etcd/etcd-backup-and-restore.md)

- etcd pod을 describe하여 확인
- 마스터노드에서 etcd 접근 host? 127.0.0.1:{port}
- Where is the ETCD server certificate file located?
    - `--cert-file=/etc/kubernetes/pki/etcd/server.crt`
- Where is the ETCD ca certificate file located?

1. snapshot 생성

```bash
# endpoint는 listen-client-urls=https://127.0.0.1:2379,https://172.17.0.38:2379
ETCDCTL_API=3 etcdctl --endpoints https://127.0.0.1:2379 \
--cacert /etc/kubernetes/pki/etcd/ca.crt \
--cert /etc/kubernetes/pki/etcd/server.crt \
--key /etc/kubernetes/pki/etcd/server.key \
snapshot save {filepath}

```

2. restore

```bash
service kube-apiserver stop

# host path의 새 etcd data dir 설정해서 restore (기존거 살리기?)
ETCDCTL_API=3 etcdctl snapshot restore snapshot.db --data-dir=/var/lib/etcd-from-backup
```

3. etcd (static pod) data-dir 수정

```bash
# ex
vi /etc/kubernetes/manifests/ectd-controlplane.yaml

# hostPath 변경 
volumes:
  - hostPath:
      path: /var/lib/etcd-from-backup
      type: DirectoryOrCreate
    name: etcd-data
```

---

## Storage

**Retain**

- pvc 삭제 → pv가 삭제되진 않으나 사용 불가

**Storage Class**

- local (no-provisioner)는 dynamic provisioning XX
- WaitForFirstConsumer → 해당 pv를 사용하는 pod이 생기기 전까지는 pending

---

## Security

- kube apiserver CN

```yaml
# subject cn
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout | grep CN
```

**docker** 

```yaml
docker ps -a 
docker logs containerID -f
```

journalctl

```bash
journalctl -u etcd.server -l
```

**CertAPI**

```bash
curl https://{masterIP}:6443 | grep certification 
> 버전 확인

# kubectl apply -f csr.yaml
kubectl certification approve {crs}
kubectl certification deny {crs}

# requestor 확인
kubectl get csr {csr}
```

**KubeConfig**

- user cert file path 확인
- cluster server = https://{masterIP}:6443

```bash
kubectl config --kubeconfig={file} use-context {context}
```

**RBAC**

- authz mode → apiserver의 authorization-mode flag 확인

```bash
kubectl auth can-i list pod --as {userName}
```

- api group 정보를 잘 입력해야 한다

```bash
kubectl api-resources
```

**Private Image Registry**

```bash
kubectl create secret docker-registry private-reg-cred --docker-server=myprivateregistry.com:5000 --docker-username=dock_user --docker-password=dock_password --docker-email=dock_user@myprivateregistry.com
```

**Security Context** 

[https://kubernetes.io/docs/tasks/configure-pod-container/security-context/](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)

```bash
kubectl exec -it ubuntu-sleeper -- date -s '19 APR 2012 11:14:00'
```

- capabilities

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-sleeper
  namespace: default
spec:
  containers:
  - command:
    - sleep
    - "4800"
    image: ubuntu
    name: ubuntu-sleeper
    securityContext:
      capabilities:
        add: ["SYS_TIME"]
```

**network policy**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: internal-policy
  namespace: default
spec:
  egress:
  - ports:
    - port: 8080
      protocol: TCP
    to:
    - podSelector:
        matchLabels:
          name: payroll
  - ports:
    - port: 3306
      protocol: TCP
    to:
    - podSelector:
        matchLabels:
          name: mysql
  podSelector:
    matchLabels:
      name: internal
  policyTypes:
  - Egress
```

---

## Networking

### basic 

- What is the network interface configured for cluster connectivity on the master node? (node-to-node communication)

```bash
kubectl get node -o wide # master node ip 확인

ifconfig | grep {masterIP}
```

```bash
ip link 
# ens3
```

- mastermac address

```bash
ip link show ens3
```

- What is the IP address of the Default Gateway?

```bash
ip route show default
```

- kube scheduler는 몇번 port

```bash
netstat -nap : 연결을 기다리는 목록과 프로그램을 보여준다
netstat -an | grep 포트번호 : 특정 포트가 사용 중에 있는지 확인 
netstat -nlpt : TCP listening 상태의 포트와 프로그램을 보여준다
```

### CNI - weave

```bash
# cni plugin 확인
ls /etc/cni/net.d/
```

---

## Kubectl

```bash
k get pv --sort-by=.spec.capacity.storage \
-o=custom-columns='NAME:.metadata.name,CAPACITY:.spec.capacity.storage'
```

```bash
k config view --kubeconfig=my-kube-config \
-o=jsonpath='{.contexts[?(@.context.user=="aws-user")]}'
```

---

## Troubleshooting

- deploy = env, command, volume, mountpath, ...
- service = endpoint가 해당 pod들로 잘 되어 있나 확인 ← sellector, port, targetport, ...
- 운영 컴포넌트가 다 있나 확인 (api, scheduler, proxy, etcd, cni, dns, controller manager,..)
- kubelet 상태 확인

### ❤️ worker 

```bash
journalctl -u kubelet -f

ps aux | grep kubelet

cat /var/lib/kubeelt/config.yaml
cat /etc/kubernetes/kubelet.conf

sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

- kubelet이 돌고 있는지 확인
- kubelet config 확인  → ca file path (worker node에서 ca file path)
- kubelet kubeconfig 확인 → api-server

### network

- cni가 잘 떠있나 확인
- kube-proxy & coreDNS 는 configMap에 있다. 마운트 패스나 파일 이름 확인
- coreDNS service가 잘 있나 확인
