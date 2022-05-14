[toc]
# 目标
通过虚拟机搭建包含3个master节点，2个node节点的k8s集群。

# 环境准备
## 公共环境
### 安装虚拟机
在vmware中安装一台ubuntu，`apt`更新软件到最新版本。
Ubuntu下载地址：https://ubuntu.com/download/server
版本号：20.04.3 LTS

### 安装k8s
安装k8s的前置条件
1. 禁用交换分区
2. 让防火墙感知到桥接网络
> 执行`lsmod | grep br_netfilter`，确认内核加载了`br_netfilter`模块
> 如果没有显示，就执行`sudo modprobe br_netfilter`显示的加载
```shell
$ lsmod | grep br_netfilter
br_netfilter           28672  0
bridge                176128  1 br_netfilter
```
> 设置内核参数
```shell
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```
3. 确认6443端口没有被占用，执行`telnet 127.0.0.1 6443`
4. 安装容器运行时，这里还是用docker
> ubuntu安装文档：https://docs.docker.com/engine/install/ubuntu/
5. 配置docker加速
```shell
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://xxxxxx.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```
6. 安装kubeadm
```shell
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### 复制虚拟机
通过vmware的克隆功能，复制出4台ubuntu。

### 配置网络

这里用表格，配置网络，hostname等相关参数 

https://blog.csdn.net/Datura_qing/article/details/122986681
