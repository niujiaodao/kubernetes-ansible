# kubernetes-ansible
### 本次安装的版本
- Kubernetes v1.13.4
- CNI v0.7.4
- Etcd v3.3.12
- Flannel v0.11.0 或者 Calico v3.4
- Docker CE 18.06.03

### 本次部署的网络信息
- Cluster IP CIDR

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
  
- 由于需要使用overlay2驱动，需要升级centos7的内核至4以上  

- docker官方的内核检查脚本建议(RHEL7/CentOS7: User namespaces disabled; add ‘user_namespace.enable=1’ to boot command line),使用下面命令开启  
  `grubby --args="user_namespace.enable=1" --update-kernel="$(grubby --default-kernel)"`

- 切记所有机器需要自行设定ntp

- 在ansible下载Kubernetes二进制文件后分发到其他机器  
  `export KUBE_VERSION=v1.13.4`
  `curl https://storage.googleapis.com/kubernetes-release/release/${KUBE_VERSION}/kubernetes-server-linux-amd64.tar.gz > kubernetes-server-linux-amd64.tar.gz`
  `tar -zxvf kubernetes-server-linux-amd64.tar.gz --strip-components=3 -C /usr/local/bin kubernetes/server/bin/kube{let,ctl,-apiserver,-controller-manager,-scheduler,-proxy}`
  
- 分发master相关组件二进制文件到其他master上
  ```
  for NODE in master01 master02 master03; do
    echo "--- $NODE ---"
    echo "scp /usr/local/bin/kube{let,ctl,-apiserver,-controller-manager,-scheduler,-proxy} $NODE:/usr/local/bin/ "
    scp /usr/local/bin/kube{let,ctl,-apiserver,-controller-manager,-scheduler,-proxy} $NODE:/usr/local/bin/ 
  done
  ```
  
- 分发node的kubernetes二进制文件到其他node上
  ```
  for NODE in node01 node02; do
    echo "--- $NODE ---"
    scp /usr/local/bin/kube{let,-proxy} $NODE:/usr/local/bin/ 
  done
  ```
  
- 在ansible下载Kubernetes CNI 二进制文件并分发
  ```
  export CNI_URL="https://github.com/containernetworking/plugins/releases/download"
  export CNI_VERSION=v0.7.4
  mkdir -p /opt/cni/bin
  wget  "${CNI_URL}/${CNI_VERSION}/cni-plugins-amd64-${CNI_VERSION}.tgz" 
  tar -zxf cni-plugins-amd64-${CNI_VERSION}.tgz -C /opt/cni/bin

  # 分发cni文件
  for NODE in master01 master02 master03 node01 node02; do
    echo "--- $NODE ---"
    ssh $NODE 'mkdir -p /opt/cni/bin'
    scp /opt/cni/bin/* $NODE:/opt/cni/bin/
  done
  ```

### 建立集群CA keys 与Certificates  
在这个部分,将需要产生多个元件的Certificates,这包含Etcd、Kubernetes 元件等,并且每个集群都会有一个根数位凭证认证机构(Root Certificate Authority)被用在认证API Server 与Kubelet 端的凭证。
