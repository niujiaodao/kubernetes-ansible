centos7升级内核版本以支持overlay2
===============================================
### centos7默认的内核可能比较低，为了使用overlay2存储引擎，需要将内核升级到4以上

用下面的命令查看当前系统的内核版本  
```
uname -sr 
# Linux 3.10.0-862.el7.x86_64
```

centos7下，我们用下面的命令升级内核
```
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org \
&& rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-2.el7.elrepo.noarch.rpm \
&& yum clean all \
&& yum --enablerepo=elrepo-kernel install kernel-ml -y \
&& grub2-set-default 0
```

至此，我们重启系统就已经完成了内核的升级，内核升级是完成了，我们就可以使用一些新特性了，比如在此老高以docker的新文件驱动overlay2为例，使用一下Linux kernel 4.0以后才支持的overlay2（Linux kernel 3.18以后才支持的叫overlayFS）。同时请确保docker的服务端版本不低于1.12，否则无法支持。
