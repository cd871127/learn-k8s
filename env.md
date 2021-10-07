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



启动

```
# systemctl enable keepalived
# systemctl start keepalived
# systemctl status keepalived
```



启动k8s



初始化k8s



# 拷贝证书到各个master节点，拷贝完自动加入集群，脚本如下，前提做好ssh免密登陆



```
# cat k8s-cluster-other-init.sh
#!/bin/bash
IPS=(192.168.65.51 192.168.65.52)
JOIN_CMD=`kubeadm token create --print-join-command 2> /dev/null`

for index in 0 1; do
  ip=${IPS[${index}]}
  ssh $ip "mkdir -p /etc/kubernetes/pki/etcd; mkdir -p ~/.kube/"
  scp /etc/kubernetes/pki/ca.crt $ip:/etc/kubernetes/pki/ca.crt
  scp /etc/kubernetes/pki/ca.key $ip:/etc/kubernetes/pki/ca.key
  scp /etc/kubernetes/pki/sa.key $ip:/etc/kubernetes/pki/sa.key
  scp /etc/kubernetes/pki/sa.pub $ip:/etc/kubernetes/pki/sa.pub
  scp /etc/kubernetes/pki/front-proxy-ca.crt $ip:/etc/kubernetes/pki/front-proxy-ca.crt
  scp /etc/kubernetes/pki/front-proxy-ca.key $ip:/etc/kubernetes/pki/front-proxy-ca.key
  scp /etc/kubernetes/admin.conf $ip:/etc/kubernetes/admin.conf
  scp /etc/kubernetes/admin.conf $ip:~/.kube/config
  scp $HOME/.kube/config $ip:$HOME/.kube/config
  ssh ${ip} "${JOIN_CMD} --control-plane"
done
```



```
# cat scp.sh
USER=root
CONTROL_PLANE_IPS="192.168.3.43 192.168.3.44"
for host in ${CONTROL_PLANE_IPS}; do
    scp /etc/kubernetes/pki/ca.crt "${USER}"@$host:
    scp /etc/kubernetes/pki/ca.key "${USER}"@$host:
    scp /etc/kubernetes/pki/sa.key "${USER}"@$host:
    scp /etc/kubernetes/pki/sa.pub "${USER}"@$host:
    scp /etc/kubernetes/pki/front-proxy-ca.crt "${USER}"@$host:
    scp /etc/kubernetes/pki/front-proxy-ca.key "${USER}"@$host:
    scp /etc/kubernetes/pki/etcd/ca.crt "${USER}"@$host:etcd-ca.crt
    scp /etc/kubernetes/pki/etcd/ca.key "${USER}"@$host:etcd-ca.key
    scp /etc/kubernetes/admin.conf "${USER}"@$host:
    ssh ${USER}@${host} 'mkdir -p /etc/kubernetes/pki/etcd'
    ssh ${USER}@${host} 'mv /${USER}/ca.crt /etc/kubernetes/pki/'
    ssh ${USER}@${host} 'mv /${USER}/ca.key /etc/kubernetes/pki/'
    ssh ${USER}@${host} 'mv /${USER}/sa.pub /etc/kubernetes/pki/'
    ssh ${USER}@${host} 'mv /${USER}/sa.key /etc/kubernetes/pki/'
    ssh ${USER}@${host} 'mv /${USER}/front-proxy-ca.crt /etc/kubernetes/pki/'
    ssh ${USER}@${host} 'mv /${USER}/front-proxy-ca.key /etc/kubernetes/pki/'
    ssh ${USER}@${host} 'mv /${USER}/etcd-ca.crt /etc/kubernetes/pki/etcd/ca.crt'
    ssh ${USER}@${host} 'mv /${USER}/etcd-ca.key /etc/kubernetes/pki/etcd/ca.key'
    ssh ${USER}@${host} 'mv /${USER}/admin.conf /etc/kubernetes/admin.conf'
done
```



检查`get node`



加入节点



证书有效时间

cd /etc/kubernetes/pki for crt in $(find /etc/kubernetes/pki/ -name "*.crt"); do openssl x509 -in $crt -noout -dates; done 



