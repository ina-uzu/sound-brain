# security

모든 user access는 apiserver가 관리

![security%205ed55613d3b542d2932bfe9e84af9aef/Untitled.png](security%205ed55613d3b542d2932bfe9e84af9aef/Untitled.png)

### Basic auth

- apiserver에 user, password / token 정보가 있는 file path 설정해서 인증
- not recommended

```yaml
curl https://master-node-ip:6445/api/v1/pods -u "uzu:pass1234"
```

![security%205ed55613d3b542d2932bfe9e84af9aef/Untitled%201.png](security%205ed55613d3b542d2932bfe9e84af9aef/Untitled%201.png)

![security%205ed55613d3b542d2932bfe9e84af9aef/Untitled%202.png](security%205ed55613d3b542d2932bfe9e84af9aef/Untitled%202.png)

### Asymmetic encryption

- (symmetic encryption → encode / decode를 하나의 key로)

![security%205ed55613d3b542d2932bfe9e84af9aef/Untitled%203.png](security%205ed55613d3b542d2932bfe9e84af9aef/Untitled%203.png)

- 특정 서버에 대해 private key + public key(lock)을 세트로 생성

### ex. ssh

![security%205ed55613d3b542d2932bfe9e84af9aef/Untitled%204.png](security%205ed55613d3b542d2932bfe9e84af9aef/Untitled%204.png)

![security%205ed55613d3b542d2932bfe9e84af9aef/Untitled%205.png](security%205ed55613d3b542d2932bfe9e84af9aef/Untitled%205.png)

![security%205ed55613d3b542d2932bfe9e84af9aef/Untitled%206.png](security%205ed55613d3b542d2932bfe9e84af9aef/Untitled%206.png)

---

### TLS

![security%205ed55613d3b542d2932bfe9e84af9aef/Untitled%207.png](security%205ed55613d3b542d2932bfe9e84af9aef/Untitled%207.png)

1. 웹서버에서 private key + public key 생성
2. 사용자가 https로 해당 서버에 요청을 보내면, 서버는 위의 public key를 카피해서 준다
3. 사용자 브라우저에 있는 ca public key를 통해서 해당 서버가 신뢰할 수 있는지 검증한다 (인증서 검증)
4. 신뢰할 수 있는 서버인 경우, 브라우저에서 public key로 사용자 symmetric key를 암호화해서 서버로 전달
5. 서버는 private key로 사용자의 symmetric key를 decode
6. client와 서버는 이 symmetric key를 가지고 encode / decode 하며 통신

![security%205ed55613d3b542d2932bfe9e84af9aef/Untitled%208.png](security%205ed55613d3b542d2932bfe9e84af9aef/Untitled%208.png)

- 위조된 사이트를 만들어서 해킹되는 문제가 없도록, 공인된 ca가 승인한 인증서가 아니면 브라우저에서 경고
- issued by ~
- 브라우저에서는 어떻게 fake ca인지 판별하나?
    - ca에서 브라우저에 public key를 제공한다

![security%205ed55613d3b542d2932bfe9e84af9aef/Untitled%209.png](security%205ed55613d3b542d2932bfe9e84af9aef/Untitled%209.png)

![security%205ed55613d3b542d2932bfe9e84af9aef/Untitled%2010.png](security%205ed55613d3b542d2932bfe9e84af9aef/Untitled%2010.png)

![security%205ed55613d3b542d2932bfe9e84af9aef/Untitled%2011.png](security%205ed55613d3b542d2932bfe9e84af9aef/Untitled%2011.png)

[https://m.blog.naver.com/alice_k106/221468341565](https://m.blog.naver.com/alice_k106/221468341565)

---

## TLS in K8s

[kubernetes-certs-checker.xlsx](security%205ed55613d3b542d2932bfe9e84af9aef/kubernetes-certs-checker.xlsx)

![security%205ed55613d3b542d2932bfe9e84af9aef/Untitled%2012.png](security%205ed55613d3b542d2932bfe9e84af9aef/Untitled%2012.png)

### server cert

- apiserver
- etcd
- kubelet

### client cert

- admin
- scheduler
- kube controller manager
- kube-proxy
- apiserver → etcd & kubelet에겐 client (server cert를 그대로 쓸 수도 있고, 따로 생성할 수도 있음)
- kubelet → apiserver 에겐 client

![security%205ed55613d3b542d2932bfe9e84af9aef/Untitled%2013.png](security%205ed55613d3b542d2932bfe9e84af9aef/Untitled%2013.png)

- ca는 server, kubelet / etcd 이렇게 2개로 설정될 수도 있음

---

## Certification Creation

### Server

1. generate private key `ca.key`
2. csr (certification signing request) `ca.csr`
3. sign cert → `ca.cert`

**ETCD**

ha를 위해 ectd를 여러개 둘 수도 있음 (peer)

![security%205ed55613d3b542d2932bfe9e84af9aef/Untitled%2014.png](security%205ed55613d3b542d2932bfe9e84af9aef/Untitled%2014.png)

**API SERVER**

![security%205ed55613d3b542d2932bfe9e84af9aef/Untitled%2015.png](security%205ed55613d3b542d2932bfe9e84af9aef/Untitled%2015.png)

- server cert
- etcd & kubelet client cert

**KUBELET**

- 각 노드마다 노드 이름으로 cert 생성 (system:node:node01)

### Client

** server crt(ca.cert)들을 다 copy해서 가지고 있어야 한다 

1. generate private key `admin.key` 
2. csr (certification signing request) `admin.csr`
3. sign cert →  이때 server ca.cert를 명시한다  → `admin.cert`

```bash
curl https://kube-apiserver:6443/api/v1/pods \
--key admin.key --cert admin.crt (--cacert ca.crt)
```

---

## Certification API

1. create csr object
2. review reqeuest
3. approve
4. share cert to new user

```bash
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: john
spec:
  groups:
  - system:authenticated
  # apenssl genrsa -out jane.key 2048
  request: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlJQ1ZqQ0NBVDRDQVFBd0VURVBNQTBHQTFVRUF3d0dZVzVuWld4aE1JSUJJakFOQmdrcWhraUc5dzBCQVFFRgpBQU9DQVE4QU1JSUJDZ0tDQVFFQTByczhJTHRHdTYxakx2dHhWTTJSVlRWMDNHWlJTWWw0dWluVWo4RElaWjBOCnR2MUZtRVFSd3VoaUZsOFEzcWl0Qm0wMUFSMkNJVXBGd2ZzSjZ4MXF3ckJzVkhZbGlBNVhwRVpZM3ExcGswSDQKM3Z3aGJlK1o2MVNrVHF5SVBYUUwrTWM5T1Nsbm0xb0R2N0NtSkZNMUlMRVI3QTVGZnZKOEdFRjJ6dHBoaUlFMwpub1dtdHNZb3JuT2wzc2lHQ2ZGZzR4Zmd4eW8ybmlneFNVekl1bXNnVm9PM2ttT0x1RVF6cXpkakJ3TFJXbWlECklmMXBMWnoyalVnald4UkhCM1gyWnVVV1d1T09PZnpXM01LaE8ybHEvZi9DdS8wYk83c0x0MCt3U2ZMSU91TFcKcW90blZtRmxMMytqTy82WDNDKzBERHk5aUtwbXJjVDBnWGZLemE1dHJRSURBUUFCb0FBd0RRWUpLb1pJaHZjTgpBUUVMQlFBRGdnRUJBR05WdmVIOGR4ZzNvK21VeVRkbmFjVmQ1N24zSkExdnZEU1JWREkyQTZ1eXN3ZFp1L1BVCkkwZXpZWFV0RVNnSk1IRmQycVVNMjNuNVJsSXJ3R0xuUXFISUh5VStWWHhsdnZsRnpNOVpEWllSTmU3QlJvYXgKQVlEdUI5STZXT3FYbkFvczFqRmxNUG5NbFpqdU5kSGxpT1BjTU1oNndLaTZzZFhpVStHYTJ2RUVLY01jSVUyRgpvU2djUWdMYTk0aEpacGk3ZnNMdm1OQUxoT045UHdNMGM1dVJVejV4T0dGMUtCbWRSeEgvbUNOS2JKYjFRQm1HCkkwYitEUEdaTktXTU0xMzhIQXdoV0tkNjVoVHdYOWl4V3ZHMkh4TG1WQzg0L1BHT0tWQW9FNkpsYWFHdTlQVmkKdjlOSjVaZlZrcXdCd0hKbzZXdk9xVlA3SVFjZmg3d0drWm89Ci0tLS0tRU5EIENFUlRJRklDQVRFIFJFUVVFU1QtLS0tLQo=
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - client auth
EOF

# approve
kubectl get csr
kubectl certificate approve john
```

![security%205ed55613d3b542d2932bfe9e84af9aef/Untitled%2016.png](security%205ed55613d3b542d2932bfe9e84af9aef/Untitled%2016.png)

---

## KubeConfig

![security%205ed55613d3b542d2932bfe9e84af9aef/Untitled%2017.png](security%205ed55613d3b542d2932bfe9e84af9aef/Untitled%2017.png)

```bash
# default로는 $HOME/.kube/config 사용
kubectl get pods —kubeconfig {configfile}
```

**KubeConfig File**

![security%205ed55613d3b542d2932bfe9e84af9aef/Untitled%2018.png](security%205ed55613d3b542d2932bfe9e84af9aef/Untitled%2018.png)

- clusters → server specification (host, cert, ...)
- contexts → 어떤 클러스터를 어떤 user가
- users → client specification

```yaml
apiVersion: v1
kind: Config
current-context: admin@my-kube-playgroud # 여기서 default context 설정
clusters:
- cluster:
    # or certificate-authority-data : ca.crt 파일 내용 (base64 encoding)
    certificate-authority: ca.crt
    server: https://my-kube-playground:6443 # master ip:6443
  name: my-kube-playgroud

contexts:
- context:
    cluster: my-kube-playgroud
    user: admin
    namespace: finance # opt
  name: admin@my-kube-playgroud

users:
- name: admin
  user:
    client-certificate: admin.crt
    client-key: admin.key
```

### update current context

```yaml
kubectl config use-context prod-user@production
```

---

## API Groups

![security%205ed55613d3b542d2932bfe9e84af9aef/Untitled%2019.png](security%205ed55613d3b542d2932bfe9e84af9aef/Untitled%2019.png)

**kubectl proxy**

```bash
kubectl proxy  
# ...  port

curl http://localhost:{port} 
```

---

## Authorization

```bash
# apiserver에 
--authoirzation-mode=Node,RBAC,Webhook # 여기 순차적으로 authz 처리가됨

--authoirzation-mode=AlwaysAllow
```

**node authorizer**

cert → system:nodes group인 애들은 privilege

**ABAC**

![security%205ed55613d3b542d2932bfe9e84af9aef/Untitled%2020.png](security%205ed55613d3b542d2932bfe9e84af9aef/Untitled%2020.png)

**RBAC**

- Role
    - apiGroups
    - resources
    - verbs
- RoleBinding
    - subjects
    - roleRef

    ```bash
    kubectl auth can-i create deployments
    > yes

    kubectl auth can-i delete nodes --as dev-user
    > no
    ```

++ Webhook, AlwaysAllow, AlwaysDeny

### Cluster Roles

- node 같은 **cluster scope의 resource들을 그룹핑해서 role 관리** → clster role / rb

![security%205ed55613d3b542d2932bfe9e84af9aef/Untitled%2021.png](security%205ed55613d3b542d2932bfe9e84af9aef/Untitled%2021.png)

---

## Image security

```yaml
image : nginx 

-> actual name = docker.io/nginx/nginx (registry / user account / img repo)
```

**Private Repository**

- docker login private-reg.io
- docker run private-rag.io/apps/exapp

→ 이걸 어떻게 pod 생성할 때 처리하느냐? secret으로 생성 & imagePullSecrete으로 등록

```bash
kubectl create secrete docker-registry regcred \ 
--docker-server=private-reg.io \
--docker-username=uzu \
--docker-password=pass1234 \
--docker-email=uzu@test.com
```

![security%205ed55613d3b542d2932bfe9e84af9aef/Untitled%2022.png](security%205ed55613d3b542d2932bfe9e84af9aef/Untitled%2022.png)

---

## Network policy

→ cni 종류에 따라 network policy 지원 xx (flannel)

![security%205ed55613d3b542d2932bfe9e84af9aef/Untitled%2023.png](security%205ed55613d3b542d2932bfe9e84af9aef/Untitled%2023.png)

- default로는 클러스터내의 모든 pod, node 간에는 all allow
- ingress / egress traffic

![security%205ed55613d3b542d2932bfe9e84af9aef/Untitled%2024.png](security%205ed55613d3b542d2932bfe9e84af9aef/Untitled%2024.png)

→ db pod에 대해서 api-prod의 3306 ingress traffic만 허용하겠다