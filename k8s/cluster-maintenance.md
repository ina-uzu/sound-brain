# cluster maintenance

```shell
# pod 다 뺴기 & 스케줄링 제외 
kubectl drain node1 --ignore-daemonsets

# 스케줄링 제외
kubectl cordon node1

# 스케줄링 활성화
kubectl uncordon node1
```
- replicaset 이 없는 (그냥 kubectl run 으로 만든 Pod)이 있는 노드를 drain 할 경우 force 옵션이 필요하다

    → 이 경우 drain하면 해당 pod은 그냥 죽는 것임 (다른 노드에서 생성 x)
    
## Upgrade

1. upgrade master nodes
    - 마스터를 내리고 업그레이드 진행
    - 마스터다 down 상태여도 사용자 앱들은 계속 돌아가는 중
    - but, 새로 앱을 올리거나 수정하는 것은 불가능 (api server xx)

2. upgrade worker nodes
    1. 모든 노드를 한번에 내리고 업그레이드 한다 → 앱 중단
    2. 노드 하나씩 업그레이드 한다 
    3. 새 버전의 노드를  추가하고, 기존 앱들을 새 노드로 차근차근 옮긴다
    
### Kubeadm upgrade
- 참고: https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/#upgrading-control-plane-nodes

```shell
# latest stable version
kubeadm upgrade plan
```

**전체 순서**
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
``` 

#### 각 노드마다 아래 단계를 수행한다

0. drain node

1. upgrade kubeadm version

```bash
# ssh {node}
apt-mark unhold kubeadm && \
apt-get update && apt-get install -y kubeadm=1.19.0-00 && \
apt-mark hold kubeadm

apt-get update && \
apt-get install -y --allow-change-held-packages kubeadm=1.19.0-00

# master node 1번 인 경우 (업그레이드 하는 첫 마스터 노드인 경우)
sudo kubeadm upgrade apply v1.19.0

# 아닌 경우 
sudo kubeadm upgrade node

# kubelet이 업그레이드 되지 않아서
# 이 상태에서 버전 확인해도 바뀌지 않음 
kubectl get nodes
```

2. upgrade kubelet & kubectl 
```bash
apt-mark unhold kubelet kubectl && \
apt-get update && apt-get install -y kubelet=1.19.0-00 kubectl=1.19.0-00 && \
apt-mark hold kubelet kubectl

apt-get update && \
apt-get install -y --allow-change-held-packages kubelet=1.19.0-00 kubectl=1.19.0-00 

# restart
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# 버전 올라간 거 확인
kubectl get nodes 
```

3. uncordon node

---

## Backup

**object files**

```bash
kubectl get all --all-namespaces -o yaml > all-deploy-services.yaml
```

### **etcd**
- 참고 : https://discuss.kubernetes.io/t/etcd-backup-and-restore-management/11019/12

1. snapshot 생성

```bash
ETCDCTL_API=3 etcdctl --endpoints https://127.0.0.1:2379 \
--cacert /etc/kubernetes/pki/etcd/ca.crt \
--cert /etc/kubernetes/pki/etcd/server.crt \
--key /etc/kubernetes/pki/etcd/server.key \
snapshot save {filename}
```
flag 값들은 `kubectl describe pod {etcd-pod-name} -n kube-system` 해서 확인할 수 있다 
- endpoint는 --listen-client-urls 
- cacert 는 --trusted-ca-file
- cert는 --cert-file
- key는 --key-file

2. restore

```bash
service kube-apiserver stop

# etcd data dir 설정해서 etcd restore 
ETCDCTL_API=3 etcdctl snapshot restore {filename} --data-dir=/var/lib/etcd-from-backup
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
