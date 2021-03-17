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
- namespace

    클러스터 내부를 논리적인 단위로 분리합니다.namespace안에 pod, controller, service, ingress등이 위치합니다.클러스터가 생성되고 나면 기본적으로 kube-system(시스템용), default(사용자용) 2개의 namespace가 만들어 집니다.

- pod

    kubernetes에서 컨테이너가 실행되는 최소단위입니다. 여러개의 컨테이너 묶음으로 띄울수 있고 하나의 pod에 속한 컨테이너들은 모두 같은 장비안에 뜹니다. pod에 속한 컨테이너들은 모두 동일한 (클러스터 내부에서 사용가능한) IP를 사용하게 되고 각각 port로 구분해서 접근할 수 있습니다.

- controller

    pod를 관리하는 역할을 합니다. ReplicaSet, ReplicationController, Deployments, StatefulSets, DaemonSet, Jobs, CronJob등이 있습니다.

    - pod자체만으로는 self-healing기능이 있지 않습니다. ReplicaSet은 선언된 pod의 개수만큼의 가용성을 보장합니다. 실제 개발 환경에서는 ReplicaSet을 직접적으로 사용하지 않고 Deployments를 통해 관리하게 됩니다.
    - 상태가 없는 컨테이너를 앱으로 띄울때는 주로 **Deployments**를 사용하게 됩니다. Deployments는 내부적으로 ReplicaSet을 만들고 이를 이용하여 pod를 관리합니다.
    - ReplicationController는 deployments가 나오기 전에 사용하던 컨트롤러로 Deployments의 사용을 권장합니다.
    - StatefulSets은 상태가 있는 앱을 띄울때 사용합니다.
    - DaemonSet은 클러스터내의 모든 노드에 동일한 pod를 띄울때 사용합니다. dkosv3에서는 로그수집기 같은걸 이걸로 띄우고 있습니다.
    - Jobs는 한번 실행하고 종료하는 작업성의 pod를 실행할때 사용합니다.
    - CronJob 은 지정된 시간에 주기적으로 실행해야하는 작업에 사용합니다. 동일한 시간대에 한꺼번에 너무 많은 CronJob을 실행하면 일부 작업이 실행되지 않는 경우도 있을수 있습니다.

- service

    pod는 클러스터내에서 계속 생성되고 삭제되고 하면서 노드를 옮겨 다니고 발급받은 IP가 변경되기도 합니다.이런 pod에 고정된 주소로 접근하기 위해서 사용하는게 service입니다.service를 사용하면 service에 연결된 pod의 변경에 상관없이 service 주소로만 접근하면 됩니다.서비스 타입은 크게 4가지가 있습니다.

    - ClusterIP : 기본 타입. 클러스터 내부에서 접근가능한 IP를 제공해 줍니다.
    - [NodePort](https://wiki.daumkakao.com/pages/viewpage.action?pageId=456997095) : 클러스터내의 모든 노드에 특정 포트를 지정된 pod에 연결해 줍니다. node1:30090 이런 식으로 접근할 수 있습니다.
    - [LoadBalancer](https://wiki.daumkakao.com/pages/viewpage.action?pageId=647499391) : 외부 로드밸런서를 pod 에 연결합니다. 접근 가능한 ExternalIP 를 부여합니다.
    - ExternalName : 클러스터 외부의 서비스로 접근할때 사용합니다. dns를 지정하면 그걸 cname으로 돌려줍니다.

- ingress
    kubernetes 클러스터 외부에서 클러스터 내부의 pod로 접근할때 사용합니다.서비스의 NodePort를 사용하면 간단하게 클러스터 외부에서 내부의 포드로 접근이 가능하지만 그보다 더 많은 기능을 이용하고 싶을때 사용합니다.

    kubernetes내부에 ingress라는 리소스를 정의해 놓으면 그 정보를 기반으로 ingress-controller가 ingress에 정의된대로 외부에서 접근가능하게 설정을 해줍니다.ingress-controller는 여러가지 종류가 있지만 dkosv3에서는 ingress-nginx-controller를 붙여놓았습니다.그래서 nginx의 다양한 기능을 사용할 수 있습니다.ingress노드를 추가하고 클러스터에 ingress를 정의해주면 nginx의 설정이 업데이트되서 반영됩니다.
