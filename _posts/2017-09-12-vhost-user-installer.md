---
layout: post
title: DPDK+OVS+QEMU搭建vhost-user实验环境
date:   2017-09-12 09:56:39
categories: virtualized-network-IO
---

目前在virtio后端驱动方面性能最好的是用户态的vhost-user，而DPDK又是用户态vhost实现里使用最广泛的。下面介绍一下怎么搭建这样一个vhost-user实验环境。我们这里使用的全部是最新的版本（ovs2.8+DPDK17.05+qemu2.9.93）.

## 1.由于涉及到虚拟化，先检查计算机是否开启了虚拟化
先确保计算机bios打开了Intel-VT，然后检查grub选项是否开启了IOMMU，vim打开/etc/default/grub，在GRUB_CMDLINE_LINUX_DEFAULT后的引号里面加上一句：

 
```
iommu=pt intel_iommu=on default_hugepagesz=1G hugepagesz=1G hugepages=8
```
如图所示：

![grub设置.png](http://upload-images.jianshu.io/upload_images/5971286-7c2f5638ac3e0d1d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这里直接添加了大页的配置，DPDK的大页配置支持2MB大页和1GB大页，使用的时候根据具体情况而定，这里我们分配了8个1GB的大页。

## 2.然后编译DPDK

DPDK默认编译成.a的静态链接库，但也可以修改DPDK文件夹下的config/common_base文件里的CONFIG_RTE_BUILD_SHARED_LIB=y （但有一点要注意的就是，common_base是生成所有平台下编译选项的基础，直接修改也会影响其他target下的编译，也可以仅仅修改生成的本平台下的config文件，或者lib库编译里面的build/.config）来编译成动态链接库。该文件是所有架构下最基础引用的配置文件，在文件中可以设置某些库不编译，以及是否编译允许debug等。

之后就和官网教程一样，设置编译目标环境，config等

 
```
make config T=x86_64-native-linuxapp-gcc

sed -ri 's,(PMD_PCAP=).*,\1y,' build/.config

make

make install
```
make完成之后，如果选择的是编译成静态链接库，这段无视。如果编译成动态链接库，去build/lib文件夹下把整个文件夹里面的.so动态链接库文件全部复制到/usr/lib/dpdk/文件夹下，没有文件夹就建一个。然后在/etc/ld.so.conf.d/文件夹下建一个dpdk.conf文件，编辑dpdk.conf，写一句话：/usr/lib/dpdk。保存退出，命令行输入ldconfig，使系统自动加载的动态链接库生效。

***也可以不这么做，但每次运行都需要改下环境变量，LD_LIBRARY_PATH指向我们刚刚编译好的lib文件夹。***

## 3.编译OVS
在ovs文件夹内configure时加上--with-dpdk选项，即可与DPDK关联。

 
```
./configure --with-dpdk=$DPDK_BUILD
```
DPDK_BUILD就是之前编译dpdk之后产生的build文件夹路径。
然后

 
```
make

make install
```
安装完成之后，就可以启动ovs了，需要DPDK实现绑定大页和网卡，以后每次启动ovs只需要最下面两个命令启动ovs最重要两个进程即可：

 
```
modprobe vfio-pci

chmod a+x /dev/vfio

chmod 0666 /dev/vfio/*

/home/yangye/ovs-dpdk/dpdk-stable-17.05.1/usertools/dpdk-devbind.py --bind=vfio-pci eth4

mount -t hugetlbfs -o pagesize=1G none /dev/hugepages

mkdir -p /usr/local/etc/openvswitch #刚装完OVS需要新创建这个目录，以后用的时候不用
ovsdb-tool create /usr/local/etc/openvswitch/conf.db vswitchd/vswitch.ovsschema  #利用ovsdb-tool创建ovsdb数据库

ovsdb-server --remote=punix:/usr/local/var/run/openvswitch/db.sock --remote=db:Open_vSwitch,Open_vSwitch,manager_options --private-key=db:Open_vSwitch,SSL,private_key --certificate=db:Open_vSwitch,SSL,certificate --bootstrap-ca-cert=db:Open_vSwitch,SSL,ca_cert --pidfile --detachovs-vsctl --no-wait set Open_vSwitch . other_config:dpdk-init=true  #启动ovsdb进程

ovs-vswitchd --pidfile --detach   #启动ovs-vswitch进程
```
其中的网卡绑定结合实际选择网卡，这样ovsdb和ovs-switchd进程都启动了，就可以执行ovs命令了。
建立一个虚拟网桥用于和qemu前端进行socket通信。

 
```
ovs-vsctl add-br br0 -- set bridge br0 datapath_type=netdev

ovs-vsctl add-port br0 vhost-user-1 -- set Interface vhost-user-1 type=dpdkvhostuserclient options:vhost-server-path="/tmp/sock0"
```
***这步可能有各种错误，耐心找下都是什么原因，大多数都是和内存有关，比如第一次有可能因为未来得及分配内存建立端口时候不成功，删除了这个端口再建一次***：


建立网桥和端口有没有成功，可以用ovs-vsctl show命令查看，正确的状态如下：

 
```
d9fd8d79-b6c0-4f13-8369-4755085471ba

   Bridge "br0"

       Port "br0"

           Interface "br0"

               type: internal

       Port "vhost-user-1"

           Interface "vhost-user-1"

               type: dpdkvhostuserclient

               options: {vhost-server-path="/tmp/sock0"}
```

至此，后端驱动vhost以及它之上的交换机已经启动了，处于等待状态。（但是qemu 2.7以上才支持重连功能）

## 4.再来安装qemu
qemu安装比较简单，直接configure、make、make install即可。
创建镜像，必须要qcow2格式：

 
```
qemu-img create -f qcow2 virtual.img 20G
```
安装虚拟机，设置了端口50，可以用VNC远处连接虚拟机：

 
```
qemu-system-x86_64 -m 2048 --enable-kvm -boot d -hda /var/iso/virtual.img -cdrom /var/iso/ubuntu-16.04.2-desktop-amd64.iso -vnc 0.0.0.0:50
```
下载一个VNC软件，像tightVNC，就可以输入"主机ip:50"来连接该虚拟机，然后进入安装流程，安装完毕后退出。

使用vhost-user作为网络接口，启动虚拟机：

 
```
qemu-system-x86_64 -machine accel=kvm -cpu host -smp sockets=2,cores=2,threads=2 -m 2048M -object memory-backend-file,id=mem,size=2048M,mem-path=/dev/hugepages,share=on -drive file=/var/iso/virtual.img -drive file=/opt/share.img,if=virtio -mem-prealloc -numa node,memdev=mem -vnc 0.0.0.0:50 --enable-kvm -chardev socket,id=char1,path=/tmp/sock0,server -netdev type=vhost-user,id=mynet1,chardev=char1,vhostforce -device virtio-net-pci,netdev=mynet1,id=net1,mac=00:00:00:00:00:01
```
虚拟机启动，vhost的前后端驱动也已连接。此时在/var/log/syslog中也能看到DPDK输出的日志信息，读取了前端qemu的内存布局。这样一个vhost-user环境就已经搭起来了，可以用TightVNC远程连接这个虚拟机了。