# Bootstrapping clusters with kubeadm
[https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

### Install Docker

[https://kubernetes.io/docs/setup/production-environment/container-runtimes/](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)

### Install Kubeame

모든 노드마다 ssh 접속하여 아래와 같은 명령어로 kubeadm, kubectl, kubelet 설치

```bash
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

- `kubeadm`: the command to bootstrap the cluster.
- `kubelet`: the component that runs on all of the machines in your cluster and does things like starting pods and containers.
- `kubectl`: the command line util to talk to your cluster.

### Initialize Master Node

이 단계는 모두 마스터 노드 안에서 진행한다. 

```bash
kubeadm init --apiserver-advertise-address= {master ip}
```

kubeconfig 생성

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

노드 등록된 것 확인

```bash
# 노드 등록된 것 확인
# pod network add on 설치가 안되어 있어서 NotReady 상태일 것
kubectl get nodes 
```

pod network add on 설치 (weave)

```bash
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"

# 노드 상태 확인 (Ready)
kubectl get nodes 
```

### Worker Node

```bash
# master node 에서 
kubeadm token create --print-join-command

# 위 명령어 output 복붙!
ssh node01
kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```
