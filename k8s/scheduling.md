# scheduling
### Manual

Pod에 Node name 설정한다. 

```yaml
spec:
  nodeName: node01
```

## Taint & Tolerations

이 팟이 어떤 노드에 뜰 수 있고, 어떤 노드에 뜰 수 없는지를 설정한다. 

**NOTE!** taint & toleration이 pod가 어떤 노드에 뜨게 할지를 정해주는 건 아님! (그건 node selector/affinity)



### **Taints**

- Taint → Node
- 노드A 에 에프킬라를 뿌린다 💩

**Taint Effects**

- NoSchedule : toleration이 없으면  pod이 스케쥴 되서 실행되지 않음. 기존에 실행되던 pod는 적용 x
- PreferNoSchedule: toleration이 없으면 pod를 스케쥴링 하지 않으려고 하긴 하지만 필수는 아님.
- NoExecute : 새로운 pod도 toleration이 없으면 실행되지 않게 하고, 기존에 있던 것도 toleration설정이 없으면 종료시킴

### **Tolerations**

- Tolerations → Pod
- pod A는 에프킬라에 강하다고 표시한다 → **but**, pod A는 다른 노드에 뜰 수도 있음
- 나머지 pod은 node A에 못 뜬다

```yaml
kind: Pod 
# . . .
spec:
  containers:
  - name: test
    image: test
    ports:
    - containerPort: 8080
  tolerations:
  - key: "app"
    operator: "Equal"
    value: "blue"
    effect: “NoSchedule"
```

---

## Node Selector

Pod가 어떤 노드에 뜨게할지 설정한다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent

  # (size=large) label이 있는 node 선택 
  nodeSelector:
    size: large
```

---

## Node Affinity

Pod가 어떤 노드에 뜨게할지 설정한다. → node selector로 하기 힘든 좀 더 복잡한 설정을 할 수 있다. 

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent

  # node affinity
  # size가 large or medium인 노드
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: size
            operator: In
            values:
            - Large
            - Medium
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: another-node-label-key
            operator: In
            values:
            - another-node-label-value
```

### Type of node affinity

- requiredDuringSchedulingIgnoredDuringExecution
- preferredDuringSchedulingIgnoredDuringExecution : 해당하는 노드가 없으면 무시

---
## Resource Requirements & Limit

request & limit → each container

---

## DaemonSets

모드 worker 노드에 하나씩 pod을 띄운다. 

**How does it work?**

pod마다 nodeName = {nodeName} 을 추가하여 각 노드마다 뜨게 한다. 

---

## Static Pod

kube api server를 통하지 않고(k8s control plane에 의존하지 않음 → 마스터 노드의 control plane component들을 static pod으로 띄운다), 

kubelet이  staticPodPath (ex. `/etc/kubernetes/manifest`) 여기에 있는 pod yaml 파일을 보고 pod을 생성해준다. 

#### ❤️ NOTE! staticPodPath 확인

```bash
ps -aux | grep kubelet
# --config=/var/lib/kubelet/config.yaml 확인

cat /var/lib/kubelet/config.yaml | grep staticPodPath
```


1. kubelet.service 옵션에  `—pod-manifest-path`  or
2. kubelet.config 파일에 `staticPodPath`

static pod로 띄운 후 docker ps 명령어로 확인 가능하다. 

edit 불가능 (삭제 후 다시 생성해야함)

---

## Multiple Schedulers

**Deploy additioanl scheduler**

kubeadm은 scheduler pod을 띄워서 만든다. (static pod)

- scheduler-name = xxx
- leader-elect = true / false
- lock-object-name = xxx

master가 한대인 경우 leader-elect = true는 scheduler 하나만 설정되야 함. 

여러대인 경우 leader-elect = true & lock-object-name = xxx 옵션을 설정한다. 


**User custom scheduler**

pod의 spec.schedulerName : xxx 으로 지정할 수 있다. 

`k get events -o wide`로 어떤 스케줄러가 스케줄링 했는지 확인 가능
