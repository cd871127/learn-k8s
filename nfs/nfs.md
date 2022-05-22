### 1、安装nfs

```shell
sudo apt install nfs-kernel-server
```
修改文件`/etc/exports`
重启nfs
```shell
sudo service nfs-kernel-server restart
```

### 2、
示例 https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner/tree/master/deploy

