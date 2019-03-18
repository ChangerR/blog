---
title: Openstack 镜像制作
tags:
  - openstack
categories:
  - cloud
date: 2017-04-27 16:59:19
---

### 使用libvirt制作qemu镜像
- 参考 https://docs.openstack.org/image-guide/
- CloudInit 参考 https://cloudbase.it/downloads/CloudbaseInitSetup_Stable_x64.msi

### 创建qemu镜像
``` shell
qemu-img create  -f qcow2 win2012.qcow2 20G
``` 

### 查询镜像 osinfo
``` shell
osinfo-query os
```

### 使用libvirt创建虚拟机
``` shell
virt-install --connect qemu:///system -n winserver2012 --vcpus=2 -r 2048   --cdrom $(pwd)/win2012r2.iso   --disk path=$(pwd)/virtio-win.iso,device=c
drom,perms=ro   --disk=win2012.qcow2,size=40,format=qcow2,bus=virtio,cache=none    --os-type windows --os-variant=win2k12r2   --accelerate   --netwo
rk=bridge:virbr0,model=virtio   --graphics vnc
```

使用VNC连接 [ip]:0,根据提示安装系统和软件

### 根据不同系统安装cloud-init
Windows(x64): https://cloudbase.it/downloads/CloudbaseInitSetup_Stable_x64.msi
YUM: 
``` shell
yum install -y cloud-init
```
APT: 
``` shell
apt install -y cloud-init
```

### 清理镜像(IP网卡mac地址关联等)
``` shell
virt-sysprep -d trusty
```
### 压缩并上传win2012.qcow2

qemu-img 镜像转换

- windows
 - https://cloudbase.it/downloads/qemu-img-win-x64-2_3_0.zip
- linux
 - 使用apt/yum安装qemu

VHDX，VMDK镜像转换

``` shell
qemu-img convert xxx.vhdx -O qcow2 xxx.qcow2
```

压缩qemu镜像

``` shell
qemu-img convert -c xxx.qcow2 -O qcow2 xxx_compress.qcow2
```

Windows镜像必须按照virtio驱动（Windows KVM PCI驱动）
- https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/archive-virtio/virtio-win-0.1.141-1/virtio-win.iso