# lifecycle

### Rollout Commands

```bash
kubectl rollout status deployment/myapp-deployment

kubectl rollout history deployment/myapp-deployment

kubectl rollout undo deployment/myapp-deployment
```

## Deploy Strategy

1. **Recreate** : 이전꺼 다 죽이고, 그 담에 새로 띄운다  → downtime이 생긴다 
2. **RollingUpdate** 

- recreate는 한번에 replicas = 0 으로

### RollingUpdate

**Max Unavailable → 롤링 업데이트시 사용할 수 없는 최대 파드 개수**

`.spec.strategy.rollingUpdate.maxUnavailable` 은 업데이트 프로세스 중에 사용할 수 없는 최대 파드의 수를 지정하는 선택적 필드이다. 

이 값은 절대 숫자(예: 5) 또는 의도한 파드 비율(예: 10%)이 될 수 있다. 절대 값은 반올림해서 백분율로 계산한다. 만약 `.spec.strategy.rollingUpdate.maxSurge` 가 0이면 값이 0이 될 수 없다. 기본 값은 25% 이다.

예를 들어 이 값을 30%로 설정하면 롤링업데이트 시작시 즉각 이전 레플리카셋의 크기를 의도한 파드 중 70%를 스케일 다운할 수 있다. 새 파드가 준비되면 기존 레플리카셋을 스케일 다운할 수 있으며, 업데이트 중에 항상 사용 가능한 전체 파드의 수는 의도한 파드의 수의 70% 이상이 되도록 새 레플리카셋을 스케일 업할 수 있다.

**Max Surge  → 롤링 업데이트시 ( 기존파드 + 새파드 ) 최대 개수**

`.spec.strategy.rollingUpdate.maxSurge` 는 의도한 파드의 수에 대해 생성할 수 있는 최대 파드의 수를 지정하는 선택적 필드이다. 이 값은 절대 숫자(예: 5) 또는 의도한 파드 비율(예: 10%)이 될 수 있다. `MaxUnavailable` 값이 0이면 이 값은 0이 될 수 없다. 절대 값은 반올림해서 백분율로 계산한다. 기본 값은 25% 이다.

예를 들어 이 값을 30%로 설정하면 롤링업데이트 시작시 새 레플리카셋의 크기를 즉시 조정해서 기존 및 새 파드의 전체 갯수를 의도한 파드의 130%를 넘지 않도록 한다. 기존 파드가 죽으면 새로운 래플리카셋은 스케일 업할 수 있으며, 업데이트하는 동안 항상 실행하는 총 파드의 수는 최대 의도한 파드의 수의 130%가 되도록 보장한다.

---

## Upgrade
### Rollback

ReplicaSet이 살아있어서 rollback 가능

```bash
kubectl rollout undo deploy/abc
```
---

## Command

- command는 overwrite (default parameter) / entrypoint는 append
- 만약 entrypoint도 수정하고 싶으면 docker run —entrypoint sleep2.0 ubuntu-sleeper 10

### command / arg

- docker entrypoint → command
- docker cmd → args

---

## ConfigMap

- command

```bash
# kubectl create configmap my-config --from-liternal=<key>=<value>
kubectl create configmap my-config --from-liternal=color=blue

kubectl create configmap my-config --from-file={fileName}
```

- object file

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  color: blue
```

- env

```yaml
spec:
  containers:
    - name: test
      env:
        # Define the environment variable
        - name: SPECIAL_LEVEL_KEY
          valueFrom:
            configMapKeyRef:
              # The ConfigMap containing the value you want to assign to SPECIAL_LEVEL_KEY
              name: special-config
              # Specify the key associated with the value
              key: special.how
```

---

## Secret

```bash
# --from-literl 옵션을 쓰려면 "generic"을 붙여야 한다 
kubectl create secret generic db-secret --from-literal=DB_Host=sql01 --from-literal=DB_User=root --from-literal=DB_Password=password123

kubectl create secret db-secret --from-file={fileName}
```

---

## Env

- env 하나씩 세팅하는 경우

```yaml
spec:
  containers:
    - name: test
      env:
        - name: SPECIAL_LEVEL_KEY
          valueFrom:
            configMapKeyRef:
              name: special-config
              key: special.how
        - name: SPECIAL_LEVEL_KEY_2
          valueFrom:
            secretKeyRef:
              name: special-secret
              key: special.how
       - name: key
         value: hoho
```

- 전체 env var를 secret / cm 에서 가져오는 경우

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
      - configMapRef:
          name: special-config # indent 주의
  restartPolicy: Never
```

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

---

## Init Container

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox:1.28
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']

  initContainers:
  - name: init-myservice
    image: busybox
    command: ['sh', '-c', 'git clone xxx']
  - name: init-myservice2
    image: busybox
    command: ['sh', '-c', 'git clone xxx']
```

- init container는 순차적으로 수행한다
