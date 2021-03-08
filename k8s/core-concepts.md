# core concepts

## Architecture

![core%20concepts%20c5a46924fbd64b648c6f57c30719799d/Untitled.png](core%20concepts%20c5a46924fbd64b648c6f57c30719799d/Untitled.png)

1. authenticate user
2. validate request
3. retrieve data 
4. update ETCD
5. scheduler : api server watch 하고 있다가 아직 node 할당 안된 pod 있으면 노드 할당해서 api에 알림
6. kubelet : api 서버가 해당 노드 kubelet에 알림
7. update ETCD : pod이 다 뜨면 kubelet이 api에 알려주고, api가 etcd에 status 변경함

### Controller Manager

![core%20concepts%20c5a46924fbd64b648c6f57c30719799d/Untitled%201.png](core%20concepts%20c5a46924fbd64b648c6f57c30719799d/Untitled%201.png)

kube controller manager에 포함된 애들

### Scheduler

1. filter nodes : resource 모자란 애들 등등 제끼기
2. rank nodes 

### Kubelet

1. register node
2. create pods
3. monitor node & pod

```yaml
kubeadm does not deploy kubelet
```

### Kube Proxy

pod network - internal network through all node

service가 생기면 그에 맞는 rule을 추가한다 (iptable mode에서는 iptable rule 추가) 

---