# networking

### CNI

![networking%203c7af7ea40f642bda506e0ae0e501044/Untitled.png](networking%203c7af7ea40f642bda506e0ae0e501044/Untitled.png)

- Docker has its own set of standards known as CNM → cni plugin 지정 X

![networking%203c7af7ea40f642bda506e0ae0e501044/Untitled%201.png](networking%203c7af7ea40f642bda506e0ae0e501044/Untitled%201.png)

**Weave CNI**

- agent pod을 모드 노드마다 띄운다. (ds)

![networking%203c7af7ea40f642bda506e0ae0e501044/Untitled%202.png](networking%203c7af7ea40f642bda506e0ae0e501044/Untitled%202.png)

![networking%203c7af7ea40f642bda506e0ae0e501044/Untitled%203.png](networking%203c7af7ea40f642bda506e0ae0e501044/Untitled%203.png)

---

## Pod Networking

**networking model**

![networking%203c7af7ea40f642bda506e0ae0e501044/Untitled%204.png](networking%203c7af7ea40f642bda506e0ae0e501044/Untitled%204.png)

**same node**

![networking%203c7af7ea40f642bda506e0ae0e501044/Untitled%205.png](networking%203c7af7ea40f642bda506e0ae0e501044/Untitled%205.png)

**different nodes**

![networking%203c7af7ea40f642bda506e0ae0e501044/Untitled%206.png](networking%203c7af7ea40f642bda506e0ae0e501044/Untitled%206.png)

- better way (노드마다 route 추가하지 말고, 전체 노드 통틀어서)

![networking%203c7af7ea40f642bda506e0ae0e501044/Untitled%207.png](networking%203c7af7ea40f642bda506e0ae0e501044/Untitled%207.png)

---

## Service Networking

![networking%203c7af7ea40f642bda506e0ae0e501044/Untitled%208.png](networking%203c7af7ea40f642bda506e0ae0e501044/Untitled%208.png)

- kube-proxy는 apiserver로 부터 service를 계속 watch하고 있음
- k8s가 새로운 service ip 부여 (apiserver에 service ip range 설정)
- 새로운 service가 생기면 그에 맞는 rule을 추가함

**kube-proxy**

- proxy-mode = iptables, ipvs

![networking%203c7af7ea40f642bda506e0ae0e501044/Untitled%209.png](networking%203c7af7ea40f642bda506e0ae0e501044/Untitled%209.png)

```bash
iptables -L -t net | grep {serviceName}
```

---

## DNS in k8s

![networking%203c7af7ea40f642bda506e0ae0e501044/Untitled%2010.png](networking%203c7af7ea40f642bda506e0ae0e501044/Untitled%2010.png)

![networking%203c7af7ea40f642bda506e0ae0e501044/Untitled%2011.png](networking%203c7af7ea40f642bda506e0ae0e501044/Untitled%2011.png)

### **CoreDNS**

pod/svc 가 생길 때마다 dns server에 추가된다. 

![networking%203c7af7ea40f642bda506e0ae0e501044/Untitled%2012.png](networking%203c7af7ea40f642bda506e0ae0e501044/Untitled%2012.png)

- pod
- config 파일이 필요하다 `kubectl get cm -n kube-system coredns`
- kubelet에 dns server ip가 설정됨
- search domain 은 service에 대해서만 되어있음
