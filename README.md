# ochacafe-s5-3
Ochacafe5 #3 Kubernetes Security

デモ環境の構築方法、資材置き場です。

# Kubernetes Cluster 構築

事前に以下のスペックで仮想マシンを作成して、SSHログイン可能な状態にしておきます。

VCNのセキュリティリストで「10.0.0.0/16」TCP 全てのプロトコルを許可しておきます。

以下手順は全てrootで行います。

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
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/arm64/kubectl"
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

```sh
kubectl get nodes
```

```sh
NAME                         STATUS   ROLES                  AGE     VERSION
k8s-control-plane            Ready    control-plane,master   18m     v1.23.4
k8s-node                     Ready    <none>                 4m44s   v1.23.4
```

### リポジトリから資材のクローン

```sh
git clone https://github.com/oracle-japan/ochacafe-s5-3.git
```

# Kubernetes Security - Cluster Setup -

## NetworkPolicy

### 1.全ての送信トラフィックを許可、受信トラフィックは拒否

#### サンプルPodとNamespaceの作成

```sh
kubectl apply -f ochacafe-s5-3/networkpolicy/sample-pod.yaml
```
```sh
pod/sample-pod-1 created
pod/sample-pod-2 created
namespace/test created
pod/sample-pod-3 created
pod/sample-pod-4 created
```

```sh
kubectl get pods
```
```sh
NAME           READY   STATUS    RESTARTS   AGE
sample-pod-1   1/1     Running   0          108s
sample-pod-2   1/1     Running   0          108s
```

```sh
kubectl get pods -n test
```
```sh
NAME           READY   STATUS    RESTARTS   AGE
sample-pod-3   1/1     Running   0          2m19s
sample-pod-4   1/1     Running   0          2m19s
```

```sh
kubectl get namespaces
```
```sh
NAME               STATUS   AGE
calico-apiserver   Active   71m
calico-system      Active   72m
default            Active   74m
kube-node-lease    Active   74m
kube-public        Active   74m
kube-system        Active   74m
test               Active   2m58s
tigera-operator    Active   72m
```

#### Namespace「default」と「test」に「egress-only-allow-networkpoplicy.yaml」を適用

```sh
kubectl apply -f ochacafe-s5-3/networkpolicy/egress-only-allow-networkpolicy.yaml
```
```sh
networkpolicy.networking.k8s.io/egress-only-allow-networkpolicy created
```

```sh
kubectl apply -f ochacafe-s5-3/networkpolicy/egress-only-allow-networkpolicy.yaml -n test
```
```sh
networkpolicy.networking.k8s.io/egress-only-allow-networkpolicy created
```

各Podの状況確認

```sh
kubectl get pod -o wide
```
```sh
NAME           READY   STATUS    RESTARTS   AGE   IP                NODE              NOMINATED NODE   READINESS GATES
sample-pod-1   1/1     Running   0          13m   192.168.173.129   k8s-node-517516   <none>           <none>
sample-pod-2   1/1     Running   0          13m   192.168.173.130   k8s-node-517516   <none>           <none>
```

```sh
kubectl get pod -o wide -n test
```
```sh
NAME           READY   STATUS    RESTARTS   AGE   IP                NODE              NOMINATED NODE   READINESS GATES
sample-pod-3   1/1     Running   0          14m   192.168.173.132   k8s-node-517516   <none>           <none>
sample-pod-4   1/1     Running   0          14m   192.168.173.131   k8s-node-517516   <none>           <none>
```

※ローカルIPは自動で割り振られるため上記と異なる場合はご自身の環境に合わせてください

この状態で、「sample-pod-1」から他のPodへのアクセスは不可

```sh
kubectl exec -it sample-pod-1 -- /bin/bash
```
```sh
curl 192.168.173.130
```
```sh
curl 192.168.173.132
```
```sh
curl 192.168.173.131
```

「sample-pod-1」からexit

```sh
exit
```

### 2.「app: pod1」から「app: pod2」のみ受信許可

#### 「sp-pod-allow-networkpolicy.yaml」を適用

```sh
kubectl apply -f ochacafe-s5-3/networkpolicy/sp-pod-allow-networkpolicy.yaml
```
```sh
networkpolicy.networking.k8s.io/sp-pod-allow-networkpolicy created
```

#### 「sample-pod-1」から「sample-pod-2」へのアクセスは可、他のPodへのアクセスは不可

```sh
kubectl exec -it sample-pod-1 -- /bin/bash
```
```sh
curl 192.168.173.130
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```
```sh
curl 192.168.173.132
```
```sh
curl 192.168.173.131
```

「sample-pod-1」からexit

```sh
exit
```

### 3.Namespace defaultのPodから「app: pod3」のみ受信許可

#### 「sp-namespace-allow-networkpolicy.yaml」の適用

```sh
kubectl apply -f ochacafe-s5-3/networkpolicy/sp-namespace-allow-networkpolicy.yaml
```
```sh
networkpolicy.networking.k8s.io/sp-namespace-allow-networkpolicy created
```

#### 「sample-pod-1」から「sample-pod-2」から、「sample-pod-3」へのアクセスは可、「sample-pod-4」へのアクセスは不可

```sh
kubectl exec -it sample-pod-1 -- /bin/bash
```
```sh
curl 192.168.173.132
```
```sh
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```
```sh
curl 192.168.173.131
```

「sample-pod-1」からexit

```sh
exit
```

```sh
kubectl exec -it sample-pod-2 -- /bin/bash
```
```sh
curl 192.168.173.132
```
```sh
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

```sh
curl 192.168.173.131
```

「sample-pod-2」からexit

```sh
exit
```

NetworkPolicyの削除

```sh
kubectl delete -f ochacafe-s5-3/networkpolicy/sp-namespace-allow-networkpolicy.yaml
```
```sh
networkpolicy.networking.k8s.io/sp-namespace-allow-networkpolicy delete
```

### 4. 特定のIP Podから「app: pod3」のみ受信許可

#### 「sp-ip-allow-networkpolicy.yaml」の適用

```sh
kubectl apply -f ochacafe-s5-3/networkpolicy/sp-ip-allow-networkpolicy.yaml
```
```sh
networkpolicy.networking.k8s.io/sp-ip-allow-networkpolicy created
```

#### 「sample-pod-2」からのみ、「sample-pod-3」へのアクセスは可、それ以外へのアクセスは不可

```sh
kubectl exec -it sample-pod-1 -- /bin/bash
```
```sh
curl 192.168.173.132
```
```sh
curl 192.168.173.131
```

「sample-pod-1」からexit

```sh
exit
```

```sh
kubectl exec -it sample-pod-2 -- /bin/bash
```
```sh
curl 192.168.173.132
```
```sh
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```
```sh
curl 192.168.173.131
```

「sample-pod-2」からexit

```sh
exit
```

#### 適用したNetwork PolicyとサンプルPodを削除

Network Policyの削除

```sh
kubectl delete -f ochacafe-s5-3/networkpolicy/sp-ip-allow-networkpolicy.yaml
```
```sh
networkpolicy.networking.k8s.io "sp-ip-allow-networkpolicy" deleted
```

```sh
kubectl delete -f ochacafe-s5-3/networkpolicy/sp-pod-allow-networkpolicy.yaml
```
```sh
networkpolicy.networking.k8s.io "sp-pod-allow-networkpolicy" deleted
```

```sh
kubectl delete -f ochacafe-s5-3/networkpolicy/egress-only-allow-networkpolicy.yaml
```
```sh
networkpolicy.networking.k8s.io "egress-only-allow-networkpolicy" deleted
```

```sh
kubectl delete -f ochacafe-s5-3/networkpolicy/egress-only-allow-networkpolicy.yaml -n test
```
```sh
networkpolicy.networking.k8s.io "egress-only-allow-networkpolicy" deleted
```

```sh
kubectl get networkpolicy --all-namespaces
```
```sh
NAMESPACE          NAME              POD-SELECTOR     AGE
calico-apiserver   allow-apiserver   apiserver=true   149m
```

サンプルPodの削除

```sh
kubectl delete -f ochacafe-s5-3/networkpolicy/sample-pod.yaml
```

## CIS Benchmark

### 1.「job-master.yaml」の作成

```sh
cat ochacafe-s5-3/kube-bench/job-master.yaml
```
```sh
apiVersion: batch/v1
kind: Job
metadata:
  name: kube-bench-master
spec:
  template:
    spec:
      hostPID: true
      nodeSelector:
        node-role.kubernetes.io/master: ""
      tolerations:
        - key: node-role.kubernetes.io/master
          operator: Exists
          effect: NoSchedule
      containers:
        - name: kube-bench
          image: aquasec/kube-bench:latest
          #command: ["kube-bench", "run", "--targets", "master"]
          command: ["kube-bench"] #変更
          args: ["--version=1.23"] #追加
          volumeMounts:
            - name: var-lib-etcd
              mountPath: /var/lib/etcd
              readOnly: true
            - name: var-lib-kubelet
              mountPath: /var/lib/kubelet
              readOnly: true
            - name: var-lib-kube-scheduler
              mountPath: /var/lib/kube-scheduler
              readOnly: true
            - name: var-lib-kube-controller-manager
              mountPath: /var/lib/kube-controller-manager
              readOnly: true
            - name: etc-systemd
              mountPath: /etc/systemd
              readOnly: true
            - name: lib-systemd
              mountPath: /lib/systemd/
              readOnly: true
            - name: srv-kubernetes
              mountPath: /srv/kubernetes/
              readOnly: true
            - name: etc-kubernetes
              mountPath: /etc/kubernetes
              readOnly: true
              # /usr/local/mount-from-host/bin is mounted to access kubectl / kubelet, for auto-detecting the Kubernetes version.
              # You can omit this mount if you specify --version as part of the command.
            - name: usr-bin
              mountPath: /usr/local/mount-from-host/bin
              readOnly: true
            - name: etc-cni-netd
              mountPath: /etc/cni/net.d/
              readOnly: true
            - name: opt-cni-bin
              mountPath: /opt/cni/bin/
              readOnly: true
            - name: etc-passwd
              mountPath: /etc/passwd
              readOnly: true
            - name: etc-group
              mountPath: /etc/group
              readOnly: true
      restartPolicy: Never
      volumes:
        - name: var-lib-etcd
          hostPath:
            path: "/var/lib/etcd"
        - name: var-lib-kubelet
          hostPath:
            path: "/var/lib/kubelet"
        - name: var-lib-kube-scheduler
          hostPath:
            path: "/var/lib/kube-scheduler"
        - name: var-lib-kube-controller-manager
          hostPath:
            path: "/var/lib/kube-controller-manager"
        - name: etc-systemd
          hostPath:
            path: "/etc/systemd"
        - name: lib-systemd
          hostPath:
            path: "/lib/systemd"
        - name: srv-kubernetes
          hostPath:
            path: "/srv/kubernetes"
        - name: etc-kubernetes
          hostPath:
            path: "/etc/kubernetes"
        - name: usr-bin
          hostPath:
            path: "/usr/bin"
        - name: etc-cni-netd
          hostPath:
            path: "/etc/cni/net.d/"
        - name: opt-cni-bin
          hostPath:
            path: "/opt/cni/bin/"
        - name: etc-passwd
          hostPath:
            path: "/etc/passwd"
        - name: etc-group
          hostPath:
            path: "/etc/group"
```

### 2. kube-bench の実行

「job-master.yaml」の適用

```sh
kubectl apply -f ochacafe-s5-3/kube-bench/job-master.yaml
```
```sh
job.batch/kube-bench-master created
```

「kube-bench-master」PodのSTATUSがCompletedになるまで待つ。

```sh
kubectl get pods
```
```sh
NAME                      READY   STATUS      RESTARTS   AGE
kube-bench-master-64mwj   0/1     Completed   0          40s
```

### 3. 結果の確認

CompletedしたPodのログを確認

```sh
kubectl logs kube-bench-master-64mwj
```
```sh
・
・（省略）
・
== Remediations master ==
・
・（省略）
・
1.2.15 Follow the documentation and create Pod Security Policy objects as per your environment.
Then, edit the API server pod specification file /etc/kubernetes/manifests/kube-apiserver.yaml
on the master node and set the --enable-admission-plugins parameter to a
value that includes PodSecurityPolicy:
--enable-admission-plugins=...,PodSecurityPolicy,...
Then restart the API Server.
・
・（省略）
・
== Summary master ==
42 checks PASS
11 checks FAIL
11 checks WARN
0 checks INFO
・
・（省略）
・
== Summary etcd ==
7 checks PASS
0 checks FAIL
0 checks WARN
0 checks INFO
・
・（省略）
・
== Summary controlplane ==
0 checks PASS
0 checks FAIL
3 checks WARN
0 checks INFO
・
・（省略）
・
== Summary node ==
19 checks PASS
1 checks FAIL
3 checks WARN
0 checks INFO
・
・（省略）
・
== Summary policies ==
0 checks PASS
0 checks FAIL
26 checks WARN
0 checks INFO

== Summary total ==
68 checks PASS
12 checks FAIL
43 checks WARN
0 checks INFO
```

# Kubernetes Security - Cluster Hardening -

## RBAC

### 1.roleとrolebinding(k8s-manage)

#### 「pod-sa」というServiceAccountを作成

```sh
kubectl create serviceaccount pod-sa
```
```sh
serviceaccount/pod-sa created
```

```sh
kubectl get serviceaccounts
```
```sh
NAME      SECRETS   AGE
default   1         3h28m
pod-sa    1         9s
```

#### 「pod-role」というroleを作成

```sh
kubectl create role pod-role --resource pods --verb list -o yaml --dry-run=client > pod-role.yaml
```

```sh
cat pod-role.yaml
```
```sh
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  creationTimestamp: null
  name: pod-role
rules:
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - list
```

```sh
kubectl apply -f pod-role.yaml
```
```sh
role.rbac.authorization.k8s.io/pod-role created
```

```sh
kubectl get roles
```
```sh
NAME       CREATED AT
pod-role   2022-03-02T08:15:22Z
```

#### 「pod-rolebinding」というrolebindingの作成及び「pod-sa」というServiceAccountと紐づける

ServiceAccout pod-saでは、defaultのNamespaceでPodの一覧のみ取得が許可されているので、Podを作成しようとするとエラーとなる。

```sh
kubectl create rolebinding pod-rolebinding --role pod-role --serviceaccount default:pod-sa -o yaml --dry-run=client > pod-rolebinding.yaml
```

```sh
cat pod-rolebinding.yaml
```
```sh
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  creationTimestamp: null
  name: pod-rolebinding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-role
subjects:
- kind: ServiceAccount
  name: pod-sa
  namespace: default
```

```sh
kubectl apply -f pod-rolebinding.yaml
```
```sh
rolebinding.rbac.authorization.k8s.io/pod-rolebinding created
```

```sh
kubectl get rolebindings
```
```sh
NAME              ROLE            AGE
pod-rolebinding   Role/pod-role   45s
```

Podの一覧を取得

```sh
kubectl --as=system:serviceaccount:default:pod-sa get pods
```
```sh
NAME                      READY   STATUS      RESTARTS   AGE
kube-bench-master-64mwj   0/1     Completed   0          58m
```

Podを作成しようと試みる

```sh
kubectl --as=system:serviceaccount:default:pod-sa run nginx --image=nginx
```
```sh
Error from server (Forbidden): pods is forbidden: User "system:serviceaccount:default:pod-sa" cannot create resource "pods" in API group "" in the namespace "default"
```

### 2.clusterroleとclusterrolebinding(k8s-manage)

#### 「namespace-sa」というServiceAccountを作成

```sh
kubectl create serviceaccount namespace-sa
```
```sh
serviceaccount/namespace-sa created
```

```sh
kubectl get serviceaccounts
```
```sh
NAME           SECRETS   AGE
default        1         4h
namespace-sa   1         9s
pod-sa         1         32m
```

#### 「namespace-clusterrole」というclusterroleを作成

```sh
kubectl create clusterrole namespace-clusterrole --resource namespaces --verb list -o yaml --dry-run=client > namespace-clusterrole.yaml
```

```sh
cat namespace-clusterrole.yaml
```
```sh
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  creationTimestamp: null
  name: namespace-clusterrole
rules:
- apiGroups:
  - ""
  resources:
  - namespaces
  verbs:
  - list
```

```sh
kubectl apply -f namespace-clusterrole.yaml
```
```sh
clusterrole.rbac.authorization.k8s.io/namespace-clusterrole created
```

```sh
kubectl get clusterroles | grep namespace-clusterrole
```
```sh
namespace-clusterrole                                                  2022-03-02T08:49:55Z
```

#### 「namespace-clusterrolebinding」というclusterrolebindingの作成及び「namespace-sa」というServiceAccountと紐づける

ServiceAccout namespace-saでは、defaultのNamespaceでNamespaceの一覧のみ取得が許可されているので、Namespaceを作成しようとするとエラーとなる。

```sh
kubectl create clusterrolebinding namespace-clusterrolebinding --clusterrole namespace-clusterrole --serviceaccount default:namespace-sa -o yaml --dry-run=client > namespace-clusterrolebinding.yaml
```

```sh
cat namespace-clusterrolebinding.yaml
```
```sh
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  creationTimestamp: null
  name: namespace-clusterrolebinding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: namespace-clusterrole
subjects:
- kind: ServiceAccount
  name: namespace-sa
  namespace: default
```

```sh
kubectl apply -f namespace-clusterrolebinding.yaml
```
```sh
clusterrolebinding.rbac.authorization.k8s.io/namespace-clusterrolebinding created
```

```sh
kubectl get clusterrolebindings | grep namespace-clusterrolebinding
```
```sh
namespace-clusterrolebinding                           ClusterRole/namespace-clusterrole                                                  45s
```

Namespaceの一覧を取得

```sh
kubectl --as=system:serviceaccount:default:namespace-sa get namespaces
```
```sh
NAME               STATUS   AGE
calico-apiserver   Active   4h15m
calico-system      Active   4h16m
default            Active   4h18m
kube-node-lease    Active   4h18m
kube-public        Active   4h18m
kube-system        Active   4h18m
tigera-operator    Active   4h16m
```

Namespaceを作成しようと試みる

```sh
kubectl --as=system:serviceaccount:default:namespace-sa create namespace mynamespace
```
```sh
Error from server (Forbidden): namespaces is forbidden: User "system:serviceaccount:default:namespace-sa" cannot create resource "namespaces" in API group "" at the cluster scope
```

# Kubernetes Security - System Hardening -

## AppArmor

### 1.Nodeに接続して、apparmorが有効か確認(k8s-node)

「Y」と表示されれば有効状態

```sh
cat /sys/module/apparmor/parameters/enabled
```
```sh
Y
```

### 2.Nodeに接続して、以下のAppArmor Profileを作成(k8s-node)

ファイル書き込みを拒否するAppArmor Profileを作成

```sh
vim /etc/apparmor.d/k8s-apparmor-example-deny-write
```
```sh
#include <tunables/global>

profile k8s-apparmor-example-deny-write flags=(attach_disconnected) {
  #include <abstractions/base>

  file,

  # Deny all file writes.
  deny /** w,
}
```

### 3.apparmor_parserコマンドを実行して、AppArmorプロファイルを登録(k8s-node)

```sh
apparmor_parser -q /etc/apparmor.d/k8s-apparmor-example-deny-write
```

```sh
aa-status | grep k8s-apparmor-example-deny-write
```
```sh
   k8s-apparmor-example-deny-write
   /usr/bin/sleep (92896) k8s-apparmor-example-deny-write
```

Nodeからexit

```sh
exit
```
```sh
logout
```
```sh
exit
```
```sh
logout
Connection to 144.21.***.*** closed.
```

### 4.AppArmor Profileのアノテーションを定義したマニフェストの作成(k8s-manage)

```sh
cat ochacafe-s5-3/apparomor/hello-apparomor.yaml
```
```sh
apiVersion: v1
kind: Pod
metadata:
  name: hello-apparmor
  annotations:
    # Tell Kubernetes to apply the AppArmor profile "k8s-apparmor-example-deny-write".
    # Note that this is ignored if the Kubernetes node is not running version 1.4 or greater.
    container.apparmor.security.beta.kubernetes.io/hello: localhost/k8s-apparmor-example-deny-write
spec:
  containers:
  - name: hello
    image: busybox
    command: [ "sh", "-c", "echo 'Hello AppArmor!' && sleep 1h" ]
```

### 5.マニフェスト適用(k8s-manage)

```sh
kubectl apply -f ochacafe-s5-3/apparomor/hello-apparomor.yaml
```
```sh
pod/hello-apparmor created
```

```sh
kubectl get pods
```
```sh
NAME                      READY   STATUS      RESTARTS   AGE
hello-apparmor            1/1     Running     0          26s
kube-bench-master-64mwj   0/1     Completed   0          4h35m
```

コンテナがこのプロファイルで実際に実行されていることを確認するために、コンテナのproc attrをチェック。

```sh
kubectl exec hello-apparmor -- cat /proc/1/attr/current
```
```sh
k8s-apparmor-example-deny-write (enforce)
```

### 6.ファイルを書き込みによる動作確認(k8s-manage)

起動したPodからファイルを作成してエラーとなることを確認

```sh
kubectl exec hello-apparmor -- touch /tmp/test
```
```sh
touch: /tmp/file: Permission denied
command terminated with exit code 1
```

※プロファイルをannotationに設定せずに起動した場合は、「/tmp/test」は作成される

```sh
kubectl delete -f ochacafe-s5-3/apparomor/hello-apparomor.yaml
```
```sh
pod/hello-apparmor delete
```

## Seccomp

### 1.Nodeに接続して、seccompが有効か確認(k8s-node)

「CONFIG_SECCOMP=y」と表示されれば有効状態

```sh
grep CONFIG_SECCOMP= /boot/config-$(uname -r)
```
```sh
CONFIG_SECCOMP=y
```

### 2.seccompのprofileを作成(k8s-node)

専用ディレクトリを作成

```sh
mkdir -p /var/lib/kubelet/seccomp/profiles
```

プロファイルを作成

```sh
vim /var/lib/kubelet/seccomp/profiles/prohibit-mkdir.json
```
```sh
{
  "defaultAction": "SCMP_ACT_ALLOW",
  "syscalls": [
    {
      "names": ["mkdir","mkdirat"],
      "action": "SCMP_ACT_ERRNO"
    }
  ]
}
```

Nodeからexit

```sh
exit
```
```sh
logout
```
```sh
exit
```
```sh
logout
Connection to 144.21.***.*** closed.
```

### 3.Profileを指定したマニフェストを作成(k8s-manage)

```sh
cat ochacafe-s5-3/seccomp/prohibit-mkdir.yaml
```
```sh
apiVersion: v1
kind: Pod
metadata:
  name: prohibit-mkdir
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/prohibit-mkdir.json
  containers:
    - name: ubuntu
      image: ubuntu:21.10
      command:
        - sleep
        - infinity
```

### 4.マニフェストを適用(k8s-manage)

```sh
kubectl ochacafe-s5-3/seccomp/prohibit-mkdir.yaml
```
```sh
pod/prohibit-mkdir created
```

```sh
kubectl get pods
```
```sh
NAME                      READY   STATUS      RESTARTS   AGE
kube-bench-master-64mwj   0/1     Completed   0          6h3m
prohibit-mkdir            1/1     Running     0          16s
```

### 5.挙動確認(k8s-manage)

ディレクトリを作成を試みるがエラーとなる。

```sh
kubectl exec -it prohibit-mkdir -- mkdir test
```
```sh
mkdir: cannot create directory 'test': Operation not permitted
command terminated with exit code 1
```

```sh
kubectl delete -f ochacafe-s5-3/seccomp/prohibit-mkdir.yaml
```
```sh
pod "prohibit-mkdir" deleted
```