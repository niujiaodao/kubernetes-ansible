# kubernetes-ansible
### 节点信息
| OS | IP  | Hostname | CPU | Memory |
|-------|:-------:|:-------:|:-------:|:-------:|
|centos7.5|192.168.56.100|ansible|1|1G|
|centos7.5|192.168.56.101|master01|1|1G|
|centos7.5|192.168.56.102|master02|1|1G|
|centos7.5|192.168.56.103|master03|1|1G|
|centos7.5|192.168.56.104|node01|1|1G|
|centos7.5|192.168.56.105|node02|1|1G|

- 如下操作全部使用root用户
- 高可用一般建议大于等于3台的奇数台,我使用3台master来做高可用


### 事前准备
- 所有机器彼此网络互通，并且ansible登入其他节点为无密码登录  
  `ssh-keygen -t rsa, 在ansible主机上生成公钥`  
  `ssh-copy-id, 将公钥复制到master01,master02,master03,node01,node02  
  
- 所有防火墙与SELinux 已关闭  
  `systemctl stop firewalld.service #停止firewall`  
  `systemctl disable firewalld.service #禁止firewall开机启动`  
  `getenforce      ##如果SELinux status参数为enabled即为开启状态`  
  `编辑 /etc/sysconfig/selinux，修改SELINUX=disabled，然后 setenforce 0  
  
- Kubernetes v1.8+要求关闭系统Swap,若不关闭则需要修改kubelet设定参数( –fail-swap-on 设置为 false 来忽略 swap on),在所有机器使用以下指令关闭swap并注释掉/etc/fstab中swap的行  
  `swapoff -a && sysctl -w vm.swappiness=0`  
  `sed -ri '/^[^#]*swap/s@^@#@' /etc/fstab`  
- 安装工具  

- docker官方的内核检查脚本建议(RHEL7/CentOS7: User namespaces disabled; add ‘user_namespace.enable=1’ to boot command line),使用下面命令开启  
  `grubby --args="user_namespace.enable=1" --update-kernel="$(grubby --default-kernel)"`
