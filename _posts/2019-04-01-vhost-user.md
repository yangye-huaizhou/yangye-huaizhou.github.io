---
layout: post
title: Vhost-user详解
date:   2019-04-01 17:33:39
categories: virtualized-network-IO
---

在软件实现的网络I/O半虚拟化中，vhost-user在性能、灵活性和兼容性等方面达到了近乎完美的权衡。虽然它的提出已经过了四年多，也已经有了越来越多的新特性加入，但是万变不离其宗，那么今天就从整个vhost-user数据通路的建立过程，以及数据包传输流程等方面详细介绍下vhost-user架构，本文基于DPDK 17.11分析。

vhost-user的最好实现在DPDK的vhost库里，该库包含了完整的virtio后端逻辑，可以直接在虚拟交换机中抽象成一个端口使用。在最主流的软件虚拟交换机OVS（openvswitch）中，就可以使用DPDK库。
![vhost-user典型部署场景.png](/assets/picture/vhostuser1.png)
vhost-user最典型的应用场景如图所示，OVS为每个虚拟机创建一个vhost端口，实现virtio后端驱动逻辑，包括响应虚拟机收发数据包的请求，处理数据包拷贝等。每个VM实际上运行在一个独立的QEMU进程内，QEMU是负责对虚拟机设备模拟的，它已经整合了KVM通信模块，因此QEMU进程已经成为VM的主进程，其中包含vcpu等线程。QEMU启动的命令行参数可以选择网卡设备类型为virtio，它就会为每个VM虚拟出virtio设备，结合VM中使用的virtio驱动，构成了virtio的前端。

# 1.建立连接

前面说到VM实际上是运行在QEMU进程内的，那么VM启动的时候要想和OVS进程的vhost端口建立连接，从而实现数据包通路，就需要先建立起来一套控制信道。**这套控制信道是基于socket进程间通信，是发生在OVS进程与QEMU进程之间，而不是与VM，另外这套通信有自己的协议标准和message的格式**。在DPDK的lib/librte_vhost/vhost_user.c中可以看到所有的消息类型：
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
	[VHOST_USER_SET_SLAVE_REQ_FD]  = "VHOST_USER_SET_SLAVE_REQ_FD",
	[VHOST_USER_IOTLB_MSG]  = "VHOST_USER_IOTLB_MSG",
};
```
随着版本迭代，越来越多的feature被加进来，消息类型也越来越多，但是**控制信道最主要的功能就是：传递建立数据通路必须的数据结构；控制数据通路的开启和关闭以及重连功能；热迁移或关闭虚拟机时传递断开连接的消息。**

从虚拟机启动到数据通路建立完毕，所传递的消息都会记录在OVS日志文件中，对这些消息整理过后，实际流程如下图所示：
![vhost-user控制信道消息流.png](/assets/picture/vhostuser2.png)
左边一半为一些协商特性，尤其像后端驱动与前端驱动互相都不知道对方协议版本的时候，协商这些特性是必要的。特性协商完毕，接下来就要传递建立数据通路所必须的数据结构，主要包括传递共享内存的文件描述符和内存地址的转换关系，以及virtio中虚拟队列的状态信息。下面对这些最关键的部分一一详细解读。

### 设置共享内存

在虚拟机中，内存是由QEMU进程提前分配好的。QEMU一旦选择了使用vhost-user的方式进行网络通信，就需要配置VM的内存访问方式为共享的，具体的命令行参数在DPDK的文档中也有说明：
```
-object memory-backend-file,share=on,...
```
这意味着**虚拟机的内存必须是预先分配好的大页面且允许共享给其他进程**，具体原因在前一篇文章讲过，因为OVS和QEMU都是用户态进程，而数据包拷贝过程中，需要OVS进程里拥有访问QEMU进程里的虚拟机buffer的能力，所以VM内存必须被共享给OVS进程。

>*一些节约内存的方案，像virtio_balloon这样动态改变VM内存的方法不再可用，原因也很简单，OVS不可能一直跟着虚拟机不断改变共享内存啊，这样多费事。*

而在vhost库中，后端驱动接收控制信道来的消息，主动映射VM内存的代码如下：
```
static int vhost_user_set_mem_table(struct virtio_net *dev, struct VhostUserMsg *pmsg)
{
   ...
   mmap_size = RTE_ALIGN_CEIL(mmap_size, alignment);

   mmap_addr = mmap(NULL, mmap_size, PROT_READ | PROT_WRITE,
				 MAP_SHARED | MAP_POPULATE, fd, 0);
   ...
   reg->mmap_addr = mmap_addr;
   reg->mmap_size = mmap_size;
   reg->host_user_addr = (uint64_t)(uintptr_t)mmap_addr +
				      mmap_offset;
   ...
}
```
它使用的linux库函数```mmap()```来映射VM内存，详见linux编程手册<http://man7.org/linux/man-pages/man2/mmap.2.html>。注意到在映射内存之前它还干了一件事，设置内存对齐，这是因为mmap函数映射内存的基本单位是一个页，也就是说起始地址和大小都必须是页大小是整数倍，在大页面环境下，就是2MB或者1GB。只有对齐以后才能保证映射共享内存不出错，以及后续访存行为不会越界。

后面三行是保存地址转换关系的信息。这里涉及到几种地址转换，在vhost-user中最复杂的就是地址转换。**从QEMU进程角度看虚拟机内存有两种地址：GPA（Guest Physical Address）和QVA（QEMU Virtual Address）；从OVS进程看虚拟机内存也有两种地址GPA（Guest Physical Address）和VVA（vSwitch Virtual Address）**

**GPA是virtio最重要的地址，在virtio的标准中，用来存储数据包地址的虚拟队列virtqueue里面每项都是以GPA表示的。**但是对于操作系统而言，我们在进程内访问实际使用的都是虚拟地址，物理地址已经被屏蔽，也就是说进程只有拿到了物理地址所对应的虚拟地址才能够去访存（*我们编程使用的指针都是虚拟地址*）。

QEMU进程容易实现，毕竟作为VM主进程给VM预分配内存时就建立了QVA到GPA的映射关系。
![共享内存映射关系.png](/assets/picture/vhostuser3.png)

对于OVS进程，以上图为例，**mmap()函数返回的也是虚拟地址，是VM内存映射到OVS地址空间的起始地址（就是说我把这一块内存映射在了以mmap_addr起始的大小为1GB的空间里）。这样给OVS进程一个GPA，OVS进程可以利用地址偏移算出对应的VVA，然后实施访存。**

但是实际映射内存比图例复杂的多，可能VM内存被拆分成了几块，有些块起始的GPA不为0，这就需要在映射块中再加一个GPA偏移量，才能完整保留下来VVA与GPA之间的地址对应关系。这些对应关系是后续数据通路实现的基础。

>*还有一点技术细节值得注意的是，在socket中，一般是不可以直接传递文件描述符的，文件描述符在编程角度就是一个int型变量，直接传过去就是一个整数，这里也是做了一点技术上的trick，感兴趣的话也可以研究下*

### 设置虚拟队列信息

虚拟队列的结构由三部分构成：avail ring、used ring和desc ring。这里稍微详细说明一下这三个ring的作用与设计思想。

>传统网卡设计中只有一个环表，但是有两个指针，分别为驱动和网卡设备管理的。这就是一个典型的生产者-消费者问题，生产数据包的一方移动指针，另一方追逐，直到全部消费完。但是这么做的缺点在于，只能顺序执行，前一个描述符处理完之前，后一个只能等待。

>**但在虚拟队列中，把生产者和消费者分离开来。desc ring中存放的是真实的数据包描述符（就是数据包或其缓冲区地址），avail ring和used ring中存放的指向desc ring中项的索引。**前端驱动将生产出来的描述符放到avail ring中，后端驱动把已经消费的描述符放到used ring中（其实就是写desc ring中的索引，即序号）。这样前端驱动就可以根据used ring来回收已经使用的描述符，即使中间有描述符被占用，也不会影响被占用描述符之后的描述符的回收。另外DPDK还针对这种结构做了一种cache层面上预取的优化，使之更加高效。

在控制信道建立完共享内存以后，还需要在后端也建立与前端一样的虚拟队列数据结构。所需要的信息主要有：desc ring的项数（不同的前端驱动不同，比如DPDK virtio驱动和内核virtio驱动就不一样）、上次avail ring用到哪（这主要是针对重连或动态迁移的，第一次建立连接此项应为0）、虚拟队列三个ring的起始地址、设置通知信号。

这几项消息处理完以后，**后端驱动利用接收到的起始地址创建了一个和前端驱动一模一样的虚拟队列数据结构，并已经准备好收发数据包。**其中最后两项eventfd是用于需要通知的场景，例如：虚拟机使用内核virtio驱动，每次OVS的vhost端口往虚拟机发送数据包完成，都需要使用eventfd通知内核驱动去接收该数据包，在轮询驱动下，这些eventfd就没有意义了。

另外，```VHOST_USER_GET_VRING_BASE```是一个非常奇特的信号，只在虚拟机关机或者断开时会由QEMU发送给OVS进程，意味着断开数据通路。

# 2.数据通路处理

数据通路的实现在DPDK的lib/librte_vhost/virtio_net.c中，虽然代码看起来非常冗长，但是其中大部分都是处理各种特性以及硬件卸载功能的，主要逻辑却非常简单。

负责数据包的收发的主函数为：
```
uint16_t rte_vhost_enqueue_burst(int vid, uint16_t queue_id, struct rte_mbuf **pkts, uint16_t count)
//数据包流向 OVS 到 VM
uint16_t rte_vhost_dequeue_burst(int vid, uint16_t queue_id,
	struct rte_mempool *mbuf_pool, struct rte_mbuf **pkts, uint16_t count)
//数据包流向 VM 到 OVS
```
具体的发送过程概括来说就是，**如果OVS往VM发送数据包，对应的vhost端口去avail ring中读取可用的buffer地址，转换成VVA后，进行数据包拷贝，拷贝完成后发送eventfd通知VM；如果VM往OVS发送，则相反，从VM内的数据包缓冲区拷贝到DPDK的mbuf数据结构。**以下贴一段代码注释吧，不要管里面的iommu、iova，那些都是vhost-user的新特性，可用理解为iova就是GPA。
```
uint16_t
rte_vhost_dequeue_burst(int vid, uint16_t queue_id,
	struct rte_mempool *mbuf_pool, struct rte_mbuf **pkts, uint16_t count)
{
	struct virtio_net *dev;
	struct rte_mbuf *rarp_mbuf = NULL;
	struct vhost_virtqueue *vq;
	uint32_t desc_indexes[MAX_PKT_BURST];
	uint32_t used_idx;
	uint32_t i = 0;
	uint16_t free_entries;
	uint16_t avail_idx;

	dev = get_device(vid);   //根据vid获取dev实例
	if (!dev)
		return 0;

	if (unlikely(!is_valid_virt_queue_idx(queue_id, 1, dev->nr_vring))) {
		RTE_LOG(ERR, VHOST_DATA, "(%d) %s: invalid virtqueue idx %d.\n",
			dev->vid, __func__, queue_id);
		return 0;
	}          //检查虚拟队列id是否合法

	vq = dev->virtqueue[queue_id];      //获取该虚拟队列

	if (unlikely(rte_spinlock_trylock(&vq->access_lock) == 0))	//对该虚拟队列加锁
		return 0;

	if (unlikely(vq->enabled == 0))		//如果vq不可访问，对虚拟队列解锁退出
		goto out_access_unlock;

	vq->batch_copy_nb_elems = 0;  //批处理需要拷贝的数据包数目

	if (dev->features & (1ULL << VIRTIO_F_IOMMU_PLATFORM))
		vhost_user_iotlb_rd_lock(vq);

	if (unlikely(vq->access_ok == 0))
		if (unlikely(vring_translate(dev, vq) < 0))  //因为IOMMU导致的，要翻译iova_to_vva
			goto out;

	if (unlikely(dev->dequeue_zero_copy)) {   //零拷贝dequeue
		struct zcopy_mbuf *zmbuf, *next;
		int nr_updated = 0;

		for (zmbuf = TAILQ_FIRST(&vq->zmbuf_list);
		     zmbuf != NULL; zmbuf = next) {
			next = TAILQ_NEXT(zmbuf, next);

			if (mbuf_is_consumed(zmbuf->mbuf)) {
				used_idx = vq->last_used_idx++ & (vq->size - 1);
				update_used_ring(dev, vq, used_idx,
						 zmbuf->desc_idx);
				nr_updated += 1;

				TAILQ_REMOVE(&vq->zmbuf_list, zmbuf, next);
				restore_mbuf(zmbuf->mbuf);
				rte_pktmbuf_free(zmbuf->mbuf);
				put_zmbuf(zmbuf);
				vq->nr_zmbuf -= 1;
			}
		}

		update_used_idx(dev, vq, nr_updated);
	}

	/*
	 * Construct a RARP broadcast packet, and inject it to the "pkts"
	 * array, to looks like that guest actually send such packet.
	 *
	 * Check user_send_rarp() for more information.
	 *
	 * broadcast_rarp shares a cacheline in the virtio_net structure
	 * with some fields that are accessed during enqueue and
	 * rte_atomic16_cmpset() causes a write if using cmpxchg. This could
	 * result in false sharing between enqueue and dequeue.
	 *
	 * Prevent unnecessary false sharing by reading broadcast_rarp first
	 * and only performing cmpset if the read indicates it is likely to
	 * be set.
	 */
	//构造arp包注入到mbuf（pkt数组），看起来像是虚拟机发送的
	if (unlikely(rte_atomic16_read(&dev->broadcast_rarp) &&
			rte_atomic16_cmpset((volatile uint16_t *)
				&dev->broadcast_rarp.cnt, 1, 0))) {

		rarp_mbuf = rte_pktmbuf_alloc(mbuf_pool);    //从mempool中分配一个mbuf给arp包
		if (rarp_mbuf == NULL) {
			RTE_LOG(ERR, VHOST_DATA,
				"Failed to allocate memory for mbuf.\n");
			return 0;
		}
		//构造arp报文，构造成功返回0
		if (make_rarp_packet(rarp_mbuf, &dev->mac)) {
			rte_pktmbuf_free(rarp_mbuf);
			rarp_mbuf = NULL;
		} else {
			count -= 1;
		}
	}
	//计算有多少数据包，现在的avail的索引减去上次停止时的索引值，若没有直接释放vq锁退出
	free_entries = *((volatile uint16_t *)&vq->avail->idx) -
			vq->last_avail_idx;
	if (free_entries == 0)
		goto out;

	LOG_DEBUG(VHOST_DATA, "(%d) %s\n", dev->vid, __func__);

	/* Prefetch available and used ring */
	//预取前一次used和avail ring停止位置索引
	avail_idx = vq->last_avail_idx & (vq->size - 1);
	used_idx  = vq->last_used_idx  & (vq->size - 1);
	rte_prefetch0(&vq->avail->ring[avail_idx]);
	rte_prefetch0(&vq->used->ring[used_idx]);

	//此次接收过程的数据包数目，为待处理数据包总数、批处理数目最小值
	count = RTE_MIN(count, MAX_PKT_BURST);
	count = RTE_MIN(count, free_entries);
	LOG_DEBUG(VHOST_DATA, "(%d) about to dequeue %u buffers\n",
			dev->vid, count);

	/* Retrieve all of the head indexes first to avoid caching issues. */
	//从avail ring中取得所有数据包在desc中的索引，存在局部变量desc_indexes数组中
	for (i = 0; i < count; i++) {
		avail_idx = (vq->last_avail_idx + i) & (vq->size - 1);
		used_idx  = (vq->last_used_idx  + i) & (vq->size - 1);
		desc_indexes[i] = vq->avail->ring[avail_idx];

		if (likely(dev->dequeue_zero_copy == 0))	//若不支持dequeue零拷贝的话，直接将索引写入used ring中
			update_used_ring(dev, vq, used_idx, desc_indexes[i]);
	}

	/* Prefetch descriptor index. */
	rte_prefetch0(&vq->desc[desc_indexes[0]]);	//从desc中预取第一个要发的数据包描述符
	for (i = 0; i < count; i++) {
		struct vring_desc *desc, *idesc = NULL;
		uint16_t sz, idx;
		uint64_t dlen;
		int err;

		if (likely(i + 1 < count))
			rte_prefetch0(&vq->desc[desc_indexes[i + 1]]); //预取后一项

		if (vq->desc[desc_indexes[i]].flags & VRING_DESC_F_INDIRECT) {  //如果该项支持indirect desc，按照indirect处理
			dlen = vq->desc[desc_indexes[i]].len;
			desc = (struct vring_desc *)(uintptr_t)     //地址转换成vva
				vhost_iova_to_vva(dev, vq,
						vq->desc[desc_indexes[i]].addr,
						&dlen,
						VHOST_ACCESS_RO);
			if (unlikely(!desc))
				break;

			if (unlikely(dlen < vq->desc[desc_indexes[i]].len)) {
				/*
				 * The indirect desc table is not contiguous
				 * in process VA space, we have to copy it.
				 */
				idesc = alloc_copy_ind_table(dev, vq,
						&vq->desc[desc_indexes[i]]);
				if (unlikely(!idesc))
					break;

				desc = idesc;
			}

			rte_prefetch0(desc);   //预取数据包
			sz = vq->desc[desc_indexes[i]].len / sizeof(*desc);
			idx = 0;
		} 
		else {
			desc = vq->desc;    //desc数组
			sz = vq->size;		//size，个数
			idx = desc_indexes[i];	//desc索引项
		}

		pkts[i] = rte_pktmbuf_alloc(mbuf_pool);   //给正在处理的这个数据包分配mbuf
		if (unlikely(pkts[i] == NULL)) {	//分配mbuf失败，跳出包处理
			RTE_LOG(ERR, VHOST_DATA,
				"Failed to allocate memory for mbuf.\n");
			free_ind_table(idesc);
			break;
		}

		err = copy_desc_to_mbuf(dev, vq, desc, sz, pkts[i], idx,
					mbuf_pool);
		if (unlikely(err)) {
			rte_pktmbuf_free(pkts[i]);
			free_ind_table(idesc);
			break;
		}

		if (unlikely(dev->dequeue_zero_copy)) {
			struct zcopy_mbuf *zmbuf;

			zmbuf = get_zmbuf(vq);
			if (!zmbuf) {
				rte_pktmbuf_free(pkts[i]);
				free_ind_table(idesc);
				break;
			}
			zmbuf->mbuf = pkts[i];
			zmbuf->desc_idx = desc_indexes[i];

			/*
			 * Pin lock the mbuf; we will check later to see
			 * whether the mbuf is freed (when we are the last
			 * user) or not. If that's the case, we then could
			 * update the used ring safely.
			 */
			rte_mbuf_refcnt_update(pkts[i], 1);

			vq->nr_zmbuf += 1;
			TAILQ_INSERT_TAIL(&vq->zmbuf_list, zmbuf, next);
		}

		if (unlikely(!!idesc))
			free_ind_table(idesc);
	}
	vq->last_avail_idx += i;

	if (likely(dev->dequeue_zero_copy == 0)) {  //实际此次批处理的所有数据包内容拷贝
		do_data_copy_dequeue(vq);
		vq->last_used_idx += i;
		update_used_idx(dev, vq, i);  //更新used ring的当前索引，并eventfd通知虚拟机接收完成
	}

out:
	if (dev->features & (1ULL << VIRTIO_F_IOMMU_PLATFORM))
		vhost_user_iotlb_rd_unlock(vq);

out_access_unlock:
	rte_spinlock_unlock(&vq->access_lock);

	if (unlikely(rarp_mbuf != NULL)) {	//再次检查有arp报文需要发送的话，就加入到pkts数组首位，虚拟交换机的mac学习表就能第一时间更新
		/*
		 * Inject it to the head of "pkts" array, so that switch's mac
		 * learning table will get updated first.
		 */
		memmove(&pkts[1], pkts, i * sizeof(struct rte_mbuf *));
		pkts[0] = rarp_mbuf;
		i += 1;
	}

	return i;
}
```
在软件实现的网络功能中大量使用批处理，而且一批的数目是可以自己设置的，但是一般约定俗成批处理最多32个数据包。

# 3.OVS轮询逻辑

在OVS中有很多这样的vhost端口，DPDK加速的OVS已经实现绑核轮询这些端口了。因此一般会把众多的端口尽可能均匀地绑定到有限的CPU核上，一些厂商在实际的生产环境中如下图所示，甚至做到了以队列为单位来负载均衡。
![OVS多队列负载均衡.png](/assets/picture/vhostuser4.png)

另外针对物理端口和虚拟端口的绑核也有一些优化问题，主要是为了避免读写锁，比如：把物理网卡的端口全部绑定到一个核上，软件的虚拟端口放到另一个核上，这样收发很少在一个核上碰撞等。

OVS每个轮询核的工作也非常简单，**对于每个轮询到的端口，查看它有多少数据包需要接收，接收完这些数据包后，对它们进行查表（flow table，主要是五元组匹配，找到目的端口），然后依次调用对应端口的发送函数发送出去（就像vhost端口的rte_vhost_enqueue_burst函数），全部完成后继续轮询下一个端口。**而SLA、Qos机制通常都是在调用对应端口的发送函数之前起作用，比如：查看该端口的令牌桶是否有足够令牌发送这些数据包，如果需要限速会选择性丢掉一些数据包，然后执行发送函数。

其实不管是哪种软件实现的转发逻辑，都始终遵循在数据通路上尽可能简单，其他的机制、消息响应可以复杂多样的哲学。