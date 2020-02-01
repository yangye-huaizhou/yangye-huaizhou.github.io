---
layout: post
title: 编译SPDK遇到的问题
date:   2019-09-19 16:56:39
categories: DPDK
---

SPDK是Intel开发的存储开发组件，需要依赖DPDK的框架。
先编译好DPDK，跳转到SPDK目录，

```
./configure --with-dpdk={dpdk_install_dir}
```

直接make就可以。

但是大部分情况下会遇到以下问题：

```
.../spdk/test/spdk_cunit.h:39:25: fatal error: CUnit/Basic.h: No such file or directory
.../spdk/include/spdk_internal/lvolstore.h:41:23: fatal error: uuid/uuid.h: No such file or directory
bdev_aio.h:39:20: fatal error: libaio.h: No such file or directory
.../spdk/lib/iscsi/md5.h:40:25: fatal error: openssl/md5.h: No such file or directory
```

我们只需要把对应的库安装上就能解决这几个问题，在ubuntu下可以对应地使用这几个命令：

```
sudo apt-get install libcunit1-dev
sudo apt-get install uuid-dev
sudo apt-get install libaio-dev
sudo apt-get install libssl-dev
```

在centOS等其他发行版中，可能需要下载源码编译安装了。
