---
layout: post
title: DPDK安装配置（附脚本）及示例程序运行
date:   2017-05-01 13:50:39
categories: DPDK
---
去年曾经写过一篇在虚拟机里配置DPDK的文章，当时DPDK还没有支持ubuntu16.04，而且当时还不会shell编程，总体来说还是比较幼稚的。今天整理了一下写了这篇ubuntu16.04实体机下的DPDK配置博文。

DPDK是因特尔推出的数据平面开发组件，主要是提供了一个高效的用户态网卡驱动，改以往网卡驱动采用的中断方式，为绑定线程轮询网卡的高效模式，可以大幅提高吞吐率。DPDK最新版的压缩包可以去官网下载，<http://dpdk.org/download>. 我这里用的环境是ubuntu16.04，DPDK版本是16.11.1

在安装DPDK之前我们需要安装另一个linux下另一个跟网络数据包相关的函数库——libpcap，命令行抓包软件tcpdump也是基于这个库实现的。

##1. 安装libpcap
去官网 <http://www.tcpdump.org/#latest-releases> 下载libpcap的压缩包。我下的是libpcap-1.8.1. 
先安装依赖库m4、bison、flex：
```
sudo apt-get install m4 bison flex
```
在特权用户下，安装libpcap：
```
cd libpcap-1.8.1
./configure
make
make install
```
安装成功，但是后面安装DPDK的时候却提示找不到libpcap.so.1，因为libpcap.so.1默认安装到了/usr/local/lib下，我们做一个符号链接到/usr/lib/下即可。
```
sudo ln -s /usr/local/lib/libpcap.so.1 /usr/lib/libpcap.so.1
```

##2. 安装DPDK
将压缩包解压缩，按照下面的步骤安装：

*注意，以下操作中有些涉及特权级的，如果operation denied，加上sudo即可*

#####2.1. 配置并编译DPDK,架构为64位x86linux系统，gcc编译
```
make config T=x86_64-native-linuxapp-gcc
sed -ri 's,(PMD_PCAP=).*,\1y,' build/.config
make
```
![编译.png](http://upload-images.jianshu.io/upload_images/5971286-b31089dacb86636d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

编译过程如上图所示，其中如果有错，多数是由于依赖项，安装上依赖项就可以了。

#####2.2. 预配置大页存储
挂载了一个NUMA节点node0，最后一句命令的意思是在node0上预留了64个2MB的大页，即64*2048kB=128MB的内存
```
mkdir -p /mnt/huge
mount -t hugetlbfs nodev /mnt/huge
echo 64 > /sys/devices/system/node/node0/hugepages/hugepages-2048kB/nr_hugepages
```
查看网卡信息
```
ifconfig
```

![网卡信息.png](http://upload-images.jianshu.io/upload_images/5971286-f18558334ce562b0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从图中看出，两个网卡分别为enp3s0和wlp2s0。

#####2.3. 加载模块和绑定网卡
```
ifconfig enp3s0 down//先把有线网卡关掉，不然下一步会报错的。
modprobe uio  
insmod build/kmod/igb_uio.ko//插入uio和igb_uio模块
```
这一步有可能会报错，不能插入该模块，网上查原因是因为linux 4.3.X以后的内核机制造成的，需要重启电脑进入bios，关闭secure boot即可。
```
python tools/dpdk-devbind.py --bind=igb_uio enp3s0//绑定igb_uio驱动到enp3s0
```

使用dpdk自带脚本查看状态
```
python tools/dpdk-devbind.py --status  
```
![网卡绑定.png](http://upload-images.jianshu.io/upload_images/5971286-8d36fe79520e744e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
从图中可以看到0000:03:00.0网卡绑定的是igb_uio驱动，网卡绑定成功。

由于每次运行DPDK都需要分配大页和绑定网卡驱动，很麻烦，可以编写一个shell脚本，每次运行直接配置完成。
脚本内容如下：

```
#!/bin/bash
DEVICE="enp3s0"
DRIVER="igb_uio"

while getopts ":hd:r:" optname
do
  case "$optname" in
    "h")
      echo "   `basename ${0}`:usage:[-d device_name] [-r driver_name]"
      echo "   where device_name can be one in: {enp3s0,wlp2s0},driver_name can be one in: {igb_uio,rte_kni}"
      exit 1
      ;;
    "d")
      DEVICE=$OPTARG
      ;;
    "r")
      DRIVER=$OPTARG
      ;;
    *)
    # Should not occur
      echo "Unknown error while processing options"
      ;;
  esac
done

mkdir -p /mnt/huge
mount -t hugetlbfs nodev /mnt/huge
echo 64 > /sys/devices/system/node/node0/hugepages/hugepages-2048kB/nr_hugepages
ifconfig $DEVICE down
modprobe uio
insmod build/kmod/$DRIVER.ko
./tools/dpdk-devbind.py --bind=$DRIVER $DEVICE
```
另外，再写一个类似的反向的脚本，如果想要将网卡恢复正常驱动使用的时候可以恢复：

```
#!/bin/bash

DEVICE="0000:03:00.0"
DRIVER="alx"

while getopts ":hd:r:" optname
do
  case "$optname" in
    "h")
      echo "   `basename ${0}`:usage:[-d device_name] [-r driver_name]"
      echo "   where device_name can be one in: {0000:02:00.0,0000:03:00.0},driver_name can be one in: {alx,iwlwifi}"
      exit 1
      ;;
    "d")
      DEVICE=$OPTARG
      ;;
    "r")
      DRIVER=$OPTARG
      ;;
    *)
    # Should not occur
      echo "Unknown error while processing options"
      ;;
  esac
done


./tools/dpdk-devbind.py --bind=$DRIVER $DEVICE
```
上面的网卡分别是enp3s0和wlp2s0，它们自身的驱动分别是alx和iwlwifi，根据实际修改。另外DPDK有些网卡是不支持的，需要去官网查看哪些网卡不支持。

运行如下图：

![配置.png](http://upload-images.jianshu.io/upload_images/5971286-badb8fdfe672528d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![恢复.png](http://upload-images.jianshu.io/upload_images/5971286-1721086435e75e69.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##3. 运行示例程序：

**注意，这里演示用的是pcap的vdev，所以不需要绑定网卡，不然会报错找不到设备。当然vdev也可以按照官方教程里给的跑dpdk的vdev**
```
./build/app/testpmd -c 7 -n 3 --vdev=eth_pcap0,iface=enp3s0 --vdev=eth_pcap1,iface=enp3s0 -- -i --nb-cores=2 --nb-ports=2 --total-num-mbufs=2048
```
//参数分别为-c <核数,二进制表示，7即111，三个核> -n<存储通道数> --vdev<虚拟设备名，自己给发包网卡起的名字>,iface<真实设备id> -- 端口数，buff数


运行截图：

![示例程序运行1.png](http://upload-images.jianshu.io/upload_images/5971286-f835a43d49fabf56.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在testpmd输入start开始转发流

![示例程序运行2.png](http://upload-images.jianshu.io/upload_images/5971286-d2cceca3dffab786.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
输入stop，停止并显示这期间的流量。quit退出程序。