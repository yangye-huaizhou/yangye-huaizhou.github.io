---
layout: post
title: qemu内存管理以及vhost-user的协商机制下的前后端内存布局
date:   2017-09-12 10:19:39
categories: virtualized-network-IO
---

**概括来说，qemu和KVM在内存管理上的关系就是：在虚拟机启动时，qemu在qemu进程地址空间申请内存，即内存的申请是在用户空间完成的。通过kvm提供的API，把地址信息注册到KVM中，这样KVM中维护有虚拟机相关的slot，这些slot构成了一个完整的虚拟机物理地址空间。slot中记录了其对应的HVA，页面数、起始GPA等，利用它可以把一个GPA转化成HVA，这正是KVM维护EPT的技术核心。整个内存虚拟化可以分为两部分：qemu部分和kvm部分。qemu完成内存的申请，kvm实现内存的管理。**

![qemu与KVM内存管理的分工.png](http://upload-images.jianshu.io/upload_images/5971286-1e105a126e1431ae.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- qemu中地址空间分两部分，两个全局变量system_memory和system_IO，其中system_memory是所有memory_region的父object，他们只负责管理内存。
- 在KVM中，也有两个全局变量address_space_memory和address_space_memory_IO，与qemu中的memory_region对应，只有将HVA和GPA的对应关系注册到KVM模块的memslot，才可以生效成为EPT。


![system_memory和address_space_memory等的关系.png](http://upload-images.jianshu.io/upload_images/5971286-9b28de7bdb6207a5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

 在qemu 2.9的前端virtio和dpdk17.05的后端vhost-user构成的虚拟队列中，会率先通过socket建立连接，将qemu中virtio的内存布局传给vhost，vhost收到包（该消息机制有自己的协议，暂称为msg）后，分析其中的信息，这里面通信包含一套自己写的协议。包含以下内容，均是在刚建立连接时候传递的：
```
static const char *vhost_message_str[VHOST_USER_MAX] = {
    [VHOST_USER_NONE] = "VHOST_USER_NONE",
    [VHOST_USER_GET_FEATURES] = "VHOST_USER_GET_FEATURES",
    [VHOST_USER_SET_FEATURES] = "VHOST_USER_SET_FEATURES",
    [VHOST_USER_SET_OWNER] = "VHOST_USER_SET_OWNER",
    [VHOST_USER_RESET_OWNER] = "VHOST_USER_RESET_OWNER",
    [VHOST_USER_SET_MEM_TABLE] = "VHOST_USER_SET_MEM_TABLE",
    [VHOST_USER_SET_LOG_BASE] = "VHOST_USER_SET_LOG_BASE",
    [VHOST_USER_SET_LOG_FD] = "VHOST_USER_SET_LOG_FD",
    [VHOST_USER_SET_VRING_NUM] = "VHOST_USER_SET_VRING_NUM",
    [VHOST_USER_SET_VRING_ADDR] = "VHOST_USER_SET_VRING_ADDR",
    [VHOST_USER_SET_VRING_BASE] = "VHOST_USER_SET_VRING_BASE",
    [VHOST_USER_GET_VRING_BASE] = "VHOST_USER_GET_VRING_BASE",
    [VHOST_USER_SET_VRING_KICK] = "VHOST_USER_SET_VRING_KICK",
    [VHOST_USER_SET_VRING_CALL] = "VHOST_USER_SET_VRING_CALL",
    [VHOST_USER_SET_VRING_ERR]  = "VHOST_USER_SET_VRING_ERR",
    [VHOST_USER_GET_PROTOCOL_FEATURES]  = "VHOST_USER_GET_PROTOCOL_FEATURES",
    [VHOST_USER_SET_PROTOCOL_FEATURES]  = "VHOST_USER_SET_PROTOCOL_FEATURES",
    [VHOST_USER_GET_QUEUE_NUM]  = "VHOST_USER_GET_QUEUE_NUM",
    [VHOST_USER_SET_VRING_ENABLE]  = "VHOST_USER_SET_VRING_ENABLE",
    [VHOST_USER_SEND_RARP]  = "VHOST_USER_SEND_RARP",
    [VHOST_USER_NET_SET_MTU]  = "VHOST_USER_NET_SET_MTU",
};
```
其中我们最关心的就是vhost_user_set_mem_table:
```
static int
vhost_user_set_mem_table(struct virtio_net *dev, struct VhostUserMsg *pmsg)
{
	...
	for (i = 0; i < memory.nregions; i++) {
		fd  = pmsg->fds[i];
		reg = &dev->mem->regions[i];

		reg->guest_phys_addr = memory.regions[i].guest_phys_addr;
		reg->guest_user_addr = memory.regions[i].userspace_addr;
		reg->size            = memory.regions[i].memory_size;
		reg->fd              = fd;

		mmap_offset = memory.regions[i].mmap_offset;
		mmap_size   = reg->size + mmap_offset;

		/* mmap() without flag of MAP_ANONYMOUS, should be called
		 * with length argument aligned with hugepagesz at older
		 * longterm version Linux, like 2.6.32 and 3.2.72, or
		 * mmap() will fail with EINVAL.
		 *
		 * to avoid failure, make sure in caller to keep length
		 * aligned.
		 */
		alignment = get_blk_size(fd);
		if (alignment == (uint64_t)-1) {
			RTE_LOG(ERR, VHOST_CONFIG,
				"couldn't get hugepage size through fstat\n");
			goto err_mmap;
		}
		mmap_size = RTE_ALIGN_CEIL(mmap_size, alignment);

		mmap_addr = mmap(NULL, mmap_size, PROT_READ | PROT_WRITE,
				 MAP_SHARED | MAP_POPULATE, fd, 0);
        //对每个region调用mmap映射共享内存
		if (mmap_addr == MAP_FAILED) {
			RTE_LOG(ERR, VHOST_CONFIG,
				"mmap region %u failed.\n", i);
			goto err_mmap;
		}

	...
	return 0;

err_mmap:
	free_mem_region(dev);
	rte_free(dev->mem);
	dev->mem = NULL;
	return -1;
}
```
另外，我们在实际运行系统的过程中发现，qemu的内存布局和vhost端的内存布局，虽是通过共享内存建立的，但是既不是一整块内存映射，也不是通过零碎的region一小块一小块的映射。它们的内存布局如下：

![virtio前后端的内存布局.png](http://upload-images.jianshu.io/upload_images/5971286-190ec713c4c44f1a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在vhost这边只有两块region，而且像是将前端的内存region做了一个聚合得到的。回归代码，发现消息传递之前，传递的并非是memory_region变量，而是memory_region_section，在qemu的vhost_set_memory函数中，有这样一个操作：
```
if (add) {
        /* Add given mapping, merging adjacent regions if any */
        vhost_dev_assign_memory(dev, start_addr, size, (uintptr_t)ram);
    } else {
        /* Remove old mapping for this memory, if any. */
        vhost_dev_unassign_memory(dev, start_addr, size);
    }
```
将毗邻的memory_region合并了，这样就解释的通了。因为memory_region是一个树状结构，且有包含关系在里面，所以如果一个个传递，vhost里面用for循环进行映射到自己地址空间，效率低下，而且大多数内存vhost用不到，没有必要这么细分。