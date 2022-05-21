




证书有效时间
cd /etc/kubernetes/pki 
for crt in $(find /etc/kubernetes/pki/ -name "*.crt"); do openssl x509 -in $crt -noout -dates; done 


# 拷贝证书到各个master节点，拷贝完自动加入集群，脚本如下，前提做好ssh免密登陆
ssh-keygen -t  rsa
ssh-copy-id -i ~/.ssh/id_rsa.pub  root@192.168.0.3