# keepalived

安装keepalived

`sudo apt install keepalived ipvsadm conntrack socat`

配置文件`cat /etc/keepalived/keepalived.conf`

ip需要改

```
global_defs {
   router_id LVS_DEVEL
}

vrrp_instance VI_1 {
    state MASTER                  #声明角色，其他两台也设置MASTER
    interface ens33                  #根据自己实际的网卡名称来写
    virtual_router_id 80            #ID是唯一的，必须一致
    priority 100                          #权重100 ，根据权重来选举虚拟ip，其他两台权重不能一样
    advert_int 1
    authentication {                    #认证方式，必须统一密码
        auth_type PASS              
        auth_pass just0kk              
    }
    virtual_ipaddress { 
        192.168.65.10                   #创建一个虚拟IP
    }
}

virtual_server 192.168.65.10 6443 {     #用于k8s-maser集群注册 的虚拟地址
    delay_loop 6
    lb_algo loadbalance
    lb_kind DR
    net_mask 255.255.255.0
    persistence_timeout 0
    protocol TCP

real_server 192.168.65.50 6443 {      #后端真实的服务
        weight 1
        SSL_GET {
            url {
              path /healthz
              status_code 200
            }
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
        }
    }


real_server 192.168.65.51 6443 {
        weight 1
        SSL_GET {
            url {
              path /healthz
              status_code 200
            }
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
        }
    }


real_server 192.168.65.52 6443 {
        weight 1
        SSL_GET {
            url {
              path /healthz
              status_code 200
            }
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
        }
    }
}
```

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
  advertiseAddress: 192.168.65.201
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
  imagePullPolicy: IfNotPresent
  #修改name为hostname
  name: k8smaster01
  taints: null
---
#添加集群IP
controlPlaneEndpoint: "192.168.65.200:6443"
apiServer:
  timeoutForControlPlane: 4m0s
  ## 添加证书域名
  certSANs:
    - "k8smaster01"
    - "k8smaster02"
    - "k8smaster03"
    - "k8scluster"
    - "192.168.65.200"
    - "192.168.65.201"
    - "192.168.65.202"
    - "192.168.65.203"
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
# 添加集群的controlpanel
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

master01生成key
sudo kubeadm init phase upload-certs --upload-certs

其他节点执行
sudo kubeadm join 192.168.65.200:6443 --token abcdef.0123456789abcdef --discovery-token-ca-cert-hash sha256:b137f187d8d0dd95e6b231057267c5b5521ce656aa73629ecdc870062b7bc67f --control-plane --certificate-key=${上一步的key}


# 拷贝证书到各个master节点，拷贝完自动加入集群，脚本如下，前提做好ssh免密登陆
ssh-keygen -t  rsa
ssh-copy-id -i ~/.ssh/id_rsa.pub  root@192.168.0.3




检查`get node`



加入节点



证书有效时间
cd /etc/kubernetes/pki for crt in $(find /etc/kubernetes/pki/ -name "*.crt"); do openssl x509 -in $crt -noout -dates; done 


