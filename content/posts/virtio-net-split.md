+++
date = '2026-05-31T22:48:22+08:00'
draft = false 
title = 'virtio-net前后端数据交互'
description = 'virtio-net与vhost报文交互分析'
tags = []
categories = ['虚拟化']
+++

工作多年，经常接触到网络虚拟化、virtio-net、vhost等概念，但始终对其没有一个完整、清晰的理解。虽然看过不少相关的技术文章，但始终不如自己总结来的印象深刻。这次打算从头到尾将virtio-net相关的内容完整过一遍，希望能有所收获。
本篇为第一篇，打算先从virtio-net前端与后端的数据交互入手，看一下报文是如何在虚拟机、宿主机之间传递的。

本文沿着virtio-net发包至vhost收包至vhost还包的路径分析，尝试构建对于virtio前后端驱动报文传递的整理理解，源码基于Linux-6.6.140。

# 代码路径
guest侧：virtio-net驱动发包入口：start_xmit
host侧：vhost报文接收：handle_tx

# 数据结构
前后端数据传递的核心结构名为vring，此结构是一个前后端均可以访问的共享内存，核心成员有三个:desc、avail、used，以前端驱动的vring结构为例
```C
struct vring {
    unsigned int num;
    vring_desc_t *desc;
    vring_avail_t *avail;
    vring_used_t *used;
};
```
desc：描述符数组，与硬件网卡驱动类似，单个desc用于存放一段内存，包括地址与长度，virtio-net发包时，单个报文可能会用到多个desc，形成一个desc chain；

avail：后端可读报文ring（环形队列），前端驱动将报文组成desc chain后，将每个chain的desc 头index存入avail ring；

used：前端可取回报文ring，后端驱动从avail ring取出报文并处理完成后，将desc chain放回used ring，前端驱动取出后释放内存；

vring在前后端驱动被包含在各自的管理结构中：
前端：注意，目前6.6内核下的前后端数据交互有两种模式，split与packed，而经典的vring结构是split模式，本文只对split做分析
```C
	struct vring_virtqueue_split {
    /* Actual memory layout for this queue. */
    struct vring vring;   //vring在这里

    /* Last written value to avail->flags */
    u16 avail_flags_shadow;

    /*
     * Last written value to avail->idx in
     * guest byte order.
     */
    u16 avail_idx_shadow;

    /* Per-descriptor state. */
    struct vring_desc_state_split *desc_state;
    struct vring_desc_extra *desc_extra;

    /* DMA address and size information */
    dma_addr_t queue_dma_addr;
    size_t queue_size_in_bytes;

    /*
     * The parameters for creating vrings are reserved for creating new
     * vring.
     */
    u32 vring_align;
    bool may_reduce_num;
};
```

后端：vring没有被定义成一个结构，而是直接包含了desc、avail、used
```c
/* The virtqueue structure describes a queue attached to a device. */
struct vhost_virtqueue {
    struct vhost_dev *dev;
    struct vhost_worker __rcu *worker;

    /* The actual ring of buffers. */
    struct mutex mutex;
    unsigned int num;
    vring_desc_t __user *desc;
    vring_avail_t __user *avail;
    vring_used_t __user *used;
    const struct vhost_iotlb_map *meta_iotlb[VHOST_NUM_ADDRS];
    struct file *kick;
    struct vhost_vring_call call_ctx;
    struct eventfd_ctx *error_ctx;
    struct eventfd_ctx *log_ctx;

    struct vhost_poll poll;

    /* The routine to call when the Guest pings us, or timeout. */
    vhost_work_fn_t handle_kick;

    /* Last available index we saw.
     * Values are limited to 0x7fff, and the high bit is used as
     * a wrap counter when using VIRTIO_F_RING_PACKED. */
    u16 last_avail_idx;

    /* Caches available index value from user. */
    u16 avail_idx;

    /* Last index we used.
     * Values are limited to 0x7fff, and the high bit is used as
     * a wrap counter when using VIRTIO_F_RING_PACKED. */
    u16 last_used_idx;

    /* Used flags */
    u16 used_flags;

    /* Last used index value we have signalled on */
    u16 signalled_used;

    /* Last used index value we have signalled on */
    bool signalled_used_valid;

    /* Log writes to used structure. */
    bool log_used;
    u64 log_addr;

    struct iovec iov[UIO_MAXIOV];
    struct iovec iotlb_iov[64];
    struct iovec *indirect;
    struct vring_used_elem *heads;
    /* Protected by virtqueue mutex. */
    struct vhost_iotlb *umem;
    struct vhost_iotlb *iotlb;
    void *private_data;
    u64 acked_features;
    u64 acked_backend_features;
    /* Log write descriptors */
    void __user *log_base;
    struct vhost_log *log;
    struct iovec log_iov[64];

    /* Ring endianness. Defaults to legacy native endianness.
     * Set to true when starting a modern virtio device. */
    bool is_le;
#ifdef CONFIG_VHOST_CROSS_ENDIAN_LEGACY
    /* Ring endianness requested by userspace for cross-endian support. */
    bool user_be;
#endif
    u32 busyloop_timeout;
};
```

# 具体流程
## virtio-net发包start_xmit
start_xmit -> xmit_skb -> virtqueue_add_outbuf -> virtqueue_add_split （virtqueue_add_packed，本文暂不考虑，先看split模式）

sg list:
	virtio发送skb前，需要将skb转换为scatter-gather list
	virtio发包会带一个特殊的virtio hdr，需要skb有足够的headroom空间，当空间不够时，就会单独用一段内存存放header，这样报文就变成了两段，需要用scatterlist数组存下来；
```c
struct scatterlist {
    unsigned long   page_link;
    unsigned int    offset;
    unsigned int    length;
    dma_addr_t  dma_address;
};
```

virtqueue_add_split流程：
	1、取下一个可用的desc头，将上面说的sg list存入一个desc chain（当前也有可能只有一个desc）；
	2、取下一个可用的avail_idx，将desc chain的头存入avail->ring[avail_idx]，此步相当于把一个desc chain存入了avail ring的某一个位置；
	3、将desc对应的skb保存下来，便于desc用完后的回收：vq->split.desc_state[i].data，i会存入对应的desc中，释放时用i来找到对应的skb；
	4、更新desc可用头、avail idx等，到此报文写入vring的过程基本完成；

通知vhost：
	start_xmit的最后，会通知vhost取包：
	virtqueue_kick_prepare
		读后端配置，确认是否需要kick，三种模式：kick、不kick、到特定desc kick；
	virtqueue_notify 通知后端
		vq->notify()
			各层自己实现pci、mmio、vdpa等；

## vhost收 handle_tx
vhost收到通知后，被唤醒后的入口函数为handle_tx；
handle_tx:
	1、从avail->ring[avail_idx]中取desc chain的头；
	2、将desc chain存入vq->iov[] （struct vhost_virtqueue），处理报文，例如sendmsg发送等；
	3、报文处理完成后vhost_net_signal_used函数将用完的desc释放，放入used ring：将处理完的desc chain head放入used->ring[last_uesd_idx]，更新idx；
	4、vhost_signal函数通知guest；

## virtio-net回收 free_old_xmit_skbs
free_old_xmit_skbs为回收入口，而实际上有几个路径都会调用：
	1、start_xmit发包路径顺便回收
		free_old_xmit_skbs
	2、skb_xmit_done->virtnet_poll_tx->free_old_xmit_skbs（这里应该是vhost通知后的回调）

free_old_xmit_skbs -> virtqueue_get_buf -> virtqueue_get_buf_ctx_split
	1、从used->ring[last_uesd_idx]取desc head，从头desc中取到desc.id，此处的id是vq->split.desc_state[]的idx，发包时驱动在这里存了desc对应的buffer（skb）地址；
	2、取vq->split.desc_state[i].data;至此拿到skb,释放资源

# 总结
发包流程：
	virtio start_xmit -> vhost handle_tx -> virtio free_old_xmit_skbs
关键数据结构：
	1、vring 前后端共享的结构，包括desc数组、avail ring存virtio至vhost的desc head、used ring存vhost用完还回virtio的desc head；
	2、前端关键结构struct vring_virtqueue 下的struct vring_virtqueue_split 里面包含vring共享结构以及自有的一些管理状态 avail idx等等；
	3、后端关键结构struct vhost_virtqueue包含vring及自有管理内容


# 思考
desc数组看上去不是idx++连续用的， desc用完一个之后，下一个是用的desc->next。为什么？
在硬件网卡驱动中，desc数组实际上就是顺序用的，即便是单个报文对应多个desc的情况，也是按数组下标递增使用的。此处为什么不能这样设计？

virtio是一个通用框架，实际上不止包括virtio-net，还有viritio-blk、virtio-scsi等。虽然desc在提交时是顺序提交的，但实际上在desc归还时，是不一定会顺序归还的。或者说virtio协议规定不能依赖顺序归还。

当归还乱序时，会产生这种情况：desc数组会被切分会很多可用区域、不可用区域，而非连续的可用+连续不可用。 这种情况产生了问题：可用、不可用区域已经无法用一组简单的读写指针维护，如果硬要用一对读写指针维护的话，就会出现归还阻塞的情况：假设desc 3 4 5 6被释放，但desc 0 1 2还没被释放，那么此时需要等0 1 2释放后才能使用3 4 5 6。

因此为了避免这种问题，virtio实际上是把提交、归还desc做了解耦。desc ring变成了一个desc pool，需要时直接从pool里拿一个desc chain（拿空闲的，而不是按顺序拿），当desc归还时直接把desc还回池子。这种情况不管desc以什么顺序归还，都不影响前端驱动desc的使用，同时保持了vring结构的简单。


