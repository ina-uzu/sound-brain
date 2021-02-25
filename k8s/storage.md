### Volume Types

- hostPath
- nfs
- ceph, google, aws, ...

---

### EmptyDir

emptyDir 볼륨은 **Pod가 노드에 할당 될 때 처음 생성되며 Pod가 해당 노드에서 실행되는 동안 존재**한다.

이름이 emptyDir 이듯이 처음에는 비어있다. 

Pod 내의 모든 컨테이너는 동일한 emptyDir 볼륨에서 읽고 쓸 수 있음

Pod가 다시 재시작하거나 없어지면 emptyDir의 데이터도 없어진다 

**use cases**

- scratch space, for a sort algorithm for example
- when a long computation needs to be done in memory
- as a cache

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - image: my-app-image
    name: my-app
    volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
    - name: cache-volume
      emptyDir: {}
```

---

### HostPath

해당 pod뜬 node의 volume을 이용한다. 

반드시 이미 존재하는 dir/file 이어야 하며, 그렇지 않은 경우 pod이 시작할 때 같이 생성을 해주어야 한다. 

(DirectoryOrCreate)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-webserver
spec:
  containers:
  - name: test-webserver
    image: k8s.gcr.io/test-webserver:latest
    volumeMounts:
    - mountPath: /var/local/aaa
      name: mydir
    - mountPath: /var/local/aaa/1.txt
      name: myfile
  volumes:
  - name: mydir
    hostPath:
      # Ensure the file directory is created.
      path: /var/local/aaa
      type: DirectoryOrCreate
  - name: myfile
    hostPath:
      path: /var/local/aaa/1.txt
      type: FileOrCreate
```

---

### pv & pvc


```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: local-pvc
spec:
  volumeName: local-pv
  storageClassName: local-storage # pv의 storagee class와 동일
  accessModes:
    - ReadWriteOnce # pv의 accessModes와 동일해야 함
  resources:
    requests:
      storage: 500Mi # pv의 request와 동일
```

- volumeName, storageclass, accessMode, request

**Retain**

- pvc 삭제 → pv가 삭제되진 않으나 사용 불가

---

### storage class

- storageclass 통해 provisioneer를 선택할 수 있다
- provisioner : no-provisioner → dynamic provisioning 안됨

