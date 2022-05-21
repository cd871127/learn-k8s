[toc]

# k8s搭建



## 1、系统环境

> Ubuntu 22.04
>
> k8s 1.24
>
> containerd 1.6.4



## 2、安装

### 2.1、代理工具

搭建k8s过程中需要翻墙，安装tsocks配合apt安装k8s

```bash
sudo apt install tsocks
```

配置代理地址，代理需要是socks5，修改`/etc/tsocks.conf`文件，`local`需要指定代理服务器网段

### 2.2、关闭swap

临时关闭

```bash
sudo swapoff -a
```

永久关闭

修改`/etc/fstab`，注释最后一行



### 2.3、安装containerd

用docker镜像的containerd，[docker官网](https://docs.docker.com/engine/install/ubuntu/)

#### 2.3.1、安装依赖

~~~bash
sudo apt install ca-certificates curl gnupg lsb-release
~~~

#### 2.3.2、添加GPGkey


```bash
tsocks curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

#### 2.3.3、添加docker源

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

#### 2.3.4、安装containerd，并且固定版本

```bash
sudo tsocks apt-get update
sudo tsocks apt-get install containerd.io
sudo apt-mark hold containerd.io
```

#### 2.3.5、配置内核参数

```bash
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# 设置必需的 sysctl 参数，这些参数在重新启动后仍然存在。
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

# 应用 sysctl 参数而无需重新启动
sudo sysctl --system
```

#### 2.3.6、生成默认的配置
```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
```
改为systemd cgroup驱动，修改`/etc/containerd/config.toml`

```
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
```

重启containerd
```bash
sudo systemctl restart containerd
```

### 2.4 安装k8s三件套
#### 2.4.1、修改内核参数
允许 iptables 检查桥接流量

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```

#### 2.4.2、更新 `apt` 包索引并安装使用 Kubernetes `apt` 仓库所需要的包：

```shell
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
```

#### 2.4.3、下载 Google Cloud 公开签名秘钥：

```shell
sudo tsocks curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
```

#### 2.4.4、添加 Kubernetes `apt` 仓库：

```shell
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

#### 2.4.5、更新 `apt` 包索引，安装 kubelet、kubeadm 和 kubectl，并锁定其版本：

```shell
sudo tsocks apt-get update
sudo tsocks apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

#### 2.4.6、配置crictl,修改`/etc/crictl.yaml`

```yaml
runtime-endpoint: unix:///var/run/containerd/containerd.sock
image-endpoint: unix:///var/run/containerd/containerd.sock
timeout: 10
debug: false
```

#### 2.4.7、安装kubectl命令行补全插件
[插件教程](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)
```
sudo apt-get install bash-completion
source /usr/share/bash-completion/bash_completion
echo 'source <(kubectl completion bash)' >>~/.bashrc
source .bashrc
```

## 3、搭建k8s
### 3.1、前期准备
#### 3.1.1、拉取pause3.6
这个在kubelet初始化的时候会用，而且用的是k8s.gcr.io/pause， 但是init用的3.7，如果不拉3.6会因为网络问题导致初始化异常

```bash
sudo crictl pull registry.aliyuncs.com/google_containers/pause:3.6
```
#### 3.1.2、重新tag

```bash
sudo ctr -n k8s.io i tag registry.aliyuncs.com/google_containers/pause:3.6 k8s.gcr.io/pause:3.6
```



### 3.2、搭建
单机配置参考`single-init.yaml`
集群配置参考`cluster-init.yaml`

#### 3.2.1、单机搭建
生成配置
```bash
sudo kubeadm config print init-defaults > single-init.yaml
```
 初始化
```bash
sudo kubeadm init --config single-init.yaml
```

允许master调度

```
kubectl taint node k8sserver node-role.kubernetes.io/control-plane-
```



#### 3.2.2、集群搭建

| hostname    | ip             |
| ----------- | -------------- |
| k8smaster01 | 192.168.65.201 |
| k8smaster02 | 192.168.65.202 |
| k8smaster03 | 192.168.65.203 |
| k8snode01   | 192.168.65.211 |
| k8snode02   | 192.168.65.212 |

keepalived地址 192.168.65.200 hostname：k8scluster

hosts文件：
```
192.168.65.201 k8smaster01
192.168.65.202 k8smaster02
192.168.65.203 k8smaster03
192.168.65.200 k8scluster
192.168.65.211 k8snode01
192.168.65.212 k8snode02
```

##### 3.2.2.1、安装keepalived
```
sudo apt install keepalived ipvsadm conntrack socat
```
修改配置文件`cat /etc/keepalived/keepalived.conf`

##### 3.2.2.2、初始化master
生成配置
```bash
sudo kubeadm config print init-defaults > single-init.yaml
```
 初始化
```bash
sudo kubeadm init --config single-init.yaml
```

生成key

```bash
sudo kubeadm init phase upload-certs --upload-certs
```

##### 3.2.2.3、添加master

需要添加`--control-plane`和`--certificate-key`, `--certificate-key`是上一步生成的key

```
sudo kubeadm join 192.168.65.200:6443 --token abcdef.0123456789abcdef         --discovery-token-ca-cert-hash sha256:b137f187d8d0dd95e6b231057267c5b5521ce656aa73629ecdc870062b7bc67f         --control-plane --certificate-key=d0d84f0834753f27d862497a149eaafc57b6082b668ad52b795900a23b244879
```

##### 3.2.2.4、添加node
节点加入
```
sudo kubeadm token create --print-join-command 2> /dev/null
```



### 3.3、配置kubectl
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

echo 'export KUBECONFIG=$HOME/.kube/config' >>~/.bashrc
```

### 3.4、flannel插件
```
tsocks wget https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml 
```

```
kubectl apply -f kube-flannel.yml 
```

### 3.5、dashboard
下载dashboard

```
tsocks wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.5.1/aio/deploy/recommended.yaml
```
deployment.spec.template.spec.containers.args添加， 支持跳过登录

```
- --enable-skip-login
```
service中添加

```
type: NodePort
```

service.spec.port中添加

```
nodePort: 32001
```
```
kubectl apply -f dashboard.yml 
```

创建sa，绑定到管理员角色
绑定默认账号到admin角色， dashboard可以看到所有ns

```
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: dashboard-admin
  namespace: kubernetes-dashboard
---

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: dashboard-admin
    namespace: kubernetes-dashboard
```

创建token
```
kubectl create token kubernetes-dashboard
```

https://192.168.65.200:32001 登录，用token

