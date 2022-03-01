# ochacafe-s5-3
Ochacafe5 #3 Kubernetes Security

デモ環境の構築方法、資材置き場です。

# Kubernetes Cluster 構築

事前に以下のスペックで仮想マシンを作成して、SSHログイン可能な状態にしておきます。

VCNのセキュリティリストで「10.0.0.0/16」TCP 全てのプロトコルを許可しておきます。

## ControlPlane & Node 共通セットアップ

### netfilter セットアップ

```sh
modprobe br_netfilter
```

```sh
cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
```

```sh
sysctl --system
```

### iptables セットアップ

```sh
vim /etc/iptables/rules.v4
```

以下追加

```sh
-A INPUT -p tcp -m state --state NEW -m tcp --dport 80 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 443 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 6443 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 2379 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 2380 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 10250 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 10251 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 10252 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 6443 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --match multiport --dports 30000:32767 -j ACCEPT
```

以下削除

```sh
-A INPUT -j REJECT --reject-with icmp-host-prohibited
-A FORWARD -j REJECT --reject-with icmp-host-prohibited
```

ルール適用

```sh
iptables-restore < /etc/iptables/rules.v4
```

### containerd セットアップ

```sh
cat > /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF
```

```sh
modprobe overlay
```

```sh
modprobe br_netfilter
```

```sh
cat > /etc/sysctl.d/99-kubernetes-cri.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
```

```sh
sysctl --system
```

```sh
apt-get update && apt-get install -y apt-transport-https ca-certificates curl software-properties-common
```

```sh
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
```

```sh
add-apt-repository \
    "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
    $(lsb_release -cs) \
    stable"
```

```sh
apt-get update && apt-get install -y containerd
```

```sh
mkdir -p /etc/containerd
```

```sh
containerd config default | sudo tee /etc/containerd/config.toml
```

### kubeadm、kubelet、kubectl インストール

```sh
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
```

```sh
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
```

```sh
apt-get update
```

```sh
apt-get install -y kubelet kubeadm kubectl
```

```sh
apt-mark hold kubelet kubeadm kubectl
```

```sh
systemctl daemon-reload
```

```sh
systemctl restart kubelet
```

## ControlPlane セットアップ

### Kubernetes Cluster Initial

```sh
kubeadm init --pod-network-cidr=192.168.0.0/16
```

以下コマンドをNodeセットアップ完了後にNodeで実施するので、
テキストエディタ等にコマンドを保存。

```sh
kubeadm join ***.***.***.***:6443 --token xxxxxxxxxxxxxxxxxxxxxxxx --discovery-token-ca-cert-hash sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

### kubeconfig セットアップ

Control Planeで実施

```sh
mkdir -p $HOME/.kube
```

```sh
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
```

```sh
chown $(id -u):$(id -g) $HOME/.kube/config
```

### Calico インストール

Control Planeで実施

```sh
kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml
```

```sh
kubectl create -f https://docs.projectcalico.org/manifests/custom-resources.yaml
```

```sh
kubectl get nodes
```

```sh
NAME                         STATUS   ROLES                  AGE     VERSION
k8s-control-plane            Ready    control-plane,master   3m34s   v1.23.4
```

## Node セットアップ

```sh
sudo -i
```

```sh
kubeadm join ***.***.***.***:6443 --token xxxxxxxxxxxxxxxxxxxxxxxx --discovery-token-ca-cert-hash sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

ControlPlaneまたはManage Server（kubectlセットアップ後）で以下コマンドを実施

```sh
kubectl get nodes
```

```sh
NAME                         STATUS   ROLES                  AGE     VERSION
k8s-control-plane            Ready    control-plane,master   18m     v1.23.4
k8s-node                     Ready    <none>                 4m44s   v1.23.4
```

## Manage Server セットアップ

```sh
sudo -i
```

### kubectl インストール

```sh
curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"
```

```sh
chmod +x ./kubectl
```

```sh
mv ./kubectl /usr/local/bin/kubectl
```

```sh
kubectl version --client
```

```sh
mkdir -p $HOME/.kube
```

事前にControlePlaneでkubeconfigを取得しておく

```sh
vim $HOME/.kube/config
```