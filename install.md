# k8s搭建



## 环境

ubuntu 22.04

k8s 1.24

容器运行时 containerd



### 1、基础工具安装

#### 1.1、tsocks

代理工具

```bash
sudo apt install tsocks
```

配置代理`/etc/tsocks.conf`

```properties
# This is the configuration for libtsocks (transparent socks)
# Lines beginning with # and blank lines are ignored
#
# The basic idea is to specify:
#       - Local subnets - Networks that can be accessed directly without
#                         assistance from a socks server
#       - Paths - Paths are basically lists of networks and a socks server
#                 which can be used to reach these networks
#       - Default server - A socks server which should be used to access
#                          networks for which no path is available
# Much more documentation than provided in these comments can be found in
# the man pages, tsocks(8) and tsocks.conf(8)

# Local networks
# For this example this machine can directly access 192.168.0.0/255.255.255.0
# (192.168.0.*) and 10.0.0.0/255.0.0.0 (10.*)

#local的网段需要包含server
local = 192.168.2.0/255.255.255.0
local = 10.0.0.0/255.0.0.0

# Paths
# For this example this machine needs to access 150.0.0.0/255.255.0.0 as
# well as port 80 on the network 150.1.0.0/255.255.0.0 through
# the socks 5 server at 10.1.7.25 (if this machines hostname was
# "socks.hello.com" we could also specify that, unless --disable-hostnames
# was specified to ./configure).

path {
        reaches = 150.0.0.0/255.255.0.0
        reaches = 150.1.0.0:80/255.255.0.0
        server = 10.1.7.25
        server_type = 5
        default_user = delius
        default_pass = hello
}

# Default server
# For connections that aren't to the local subnets or to 150.0.0.0/255.255.0.0
# the server at 192.168.0.1 should be used (again, hostnames could be used
# too, see note above)

server = 192.168.2.199
# Server type defaults to 4 so we need to specify it as 5 for this one
server_type = 5
# The port defaults to 1080 but I've stated it here for clarity
server_port = 10808
```



关闭swap

```
sudo swapoff -a
```

修改文件`/etc/fstab`注释最后一行swap

### 2、containerd 安装

使用docker镜像中的containerd，https://docs.docker.com/engine/install/ubuntu/

安装依赖

```bash
sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```

添加GPGkey

```
tsocks curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```



添加源

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```



安装containerd，固定版本

```bash
sudo tsocks apt-get update
sudo tsocks apt-get install containerd.io
sudo apt-mark hold containerd.io
```



配置containerd

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



生产配置

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
```



改为systemd cgroup驱动

修改`/etc/containerd/config.toml`

```
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
```





重启

```bash
sudo systemctl restart containerd
```

### 3、kube三件套

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



安装

1更新 `apt` 包索引并安装使用 Kubernetes `apt` 仓库所需要的包：

```shell
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
```

2下载 Google Cloud 公开签名秘钥：

```shell
sudo tsocks curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
```

3添加 Kubernetes `apt` 仓库：

```shell
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

4更新 `apt` 包索引，安装 kubelet、kubeadm 和 kubectl，并锁定其版本：

```shell
sudo tsocks apt-get update
sudo tsocks apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```




### 4、初始化master

生成配置

```bash
sudo kubeadm config print init-defaults > init.yaml
```



修改配置

```yaml
apiVersion: kubeadm.k8s.io/v1beta3
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
#修改IP为本机IP
  advertiseAddress: 192.168.65.133
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
  imagePullPolicy: IfNotPresent
#修改name为hostname
  name: k8sserver
  taints: null
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns: {}
etcd:
  local:
    dataDir: /var/lib/etcd
#镜像从 k8s.gcr.io改为registry.aliyuncs.com/google_containers
imageRepository: registry.aliyuncs.com/google_containers
kind: ClusterConfiguration
kubernetesVersion: 1.24.0
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
#添加pod网络，这里和flannel的默认配置一样
  podSubnet: 10.244.0.0/16
scheduler: {}
---
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
#指定用systemd
cgroupDriver: systemd

```



配置crictl,修改`/etc/crictl.yaml`

```yaml
runtime-endpoint: unix:///var/run/containerd/containerd.sock
image-endpoint: unix:///var/run/containerd/containerd.sock
timeout: 10
debug: false
```



拉去pause3.6，这个在kubelet初始化的时候会用，而且用的是k8s.gcr.io/pause， 但是init用的3.7，如果不拉3.6会因为网络问题导致初始化异常

```bash
sudo crictl pull registry.aliyuncs.com/google_containers/pause:3.6
```

重新tag

```
sudo ctr -n k8s.io i tag registry.aliyuncs.com/google_containers/pause:3.6 k8s.gcr.io/pause:3.6
```



初始化

```
sudo kubeadm init --config init.yaml
```



### 5 配置kubectl

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```



修改admin文件权限 `/etc/kubernetes/admin.conf`

```
sudo chmod 755 /etc/kubernetes/admin.conf
```

添加环境变量

```
echo 'export KUBECONFIG=/etc/kubernetes/admin.conf' >>~/.bashrc
```



安装kubectl命令行补全插件https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/

```
sudo apt-get install bash-completion
source /usr/share/bash-completion/bash_completion
echo 'alias k=kubectl' >>~/.bashrc
source .bashrc
```



测试kubectl

```bash
$ kubectl get ns
NAME              STATUS   AGE
default           Active   109s
kube-node-lease   Active   111s
kube-public       Active   111s
kube-system       Active   111s
```



安装flannel

下载flannel

```
tsocks wget https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml 
```

安装

```
kubectl apply -f kube-flannel.yml 
```





