# RDMA 学习笔记

## 1. 背景

随着以太网的提速，TCP/IP 协议基于软件协议栈传输的这种方式（图1）已经无法适应高速数据传输的需求，成为网络性能进一步增长的瓶颈。应用数据在用户态和内核态之间的拷贝带来ms级的延时，协议栈对数据包的解析、寻址、校验等操作需要消耗大量CPU资源。

普通网卡的工作过程如下：先把收到的数据包缓存到系统上，数据包经过处理后，相应数据被分配到一个TCP连接。然后，接收系统再把主动提供的TCP数据同相应的应用程序联系起来，并将数据从系统缓冲区拷贝到目标存储地址。这样，制约网络速率的因素就出现了：应用通信强度不断增加和主机CPU 在内核与应用存储器间处理数据的任务繁重，使系统要不断追加主机CPU 资源，配置高效的软件并增强系统负荷管理。



新型的网络技术替代了传统的TCP/IP软件协议栈的设计，核心技术就是RDMA，全称是远程直接内存访问，实现了kernel bypass技术，数据不需要经过软件协议栈，并且不需要CPU参与寻址、解析等操作，从而提供低延时、高带宽、低CPU使用的性能优势：

- Remote：远程服务器之间的数据交换
- Direct：内核旁路技术、协议栈下发到网卡
- Memory：用户态应用虚拟内存
- Access：SEND/RECV，READ，WRITE等操作

如图2所示，从RDMA的宏观传输图中，我们可以看到，数据直接从用户态发送给网卡，再由网卡中的协议栈进行转发到达目标端用户态内存，整个过程完全旁路了内核，不需要用户态到内核态的数据拷贝，降低了延时，同时不经过软件协议栈，也就不需要CPU参与寻址等操作，减少了CPU的使用。



## 2. RDMA技术

目前主要有三种技术支持RDMA： InfiniBand， Ethernet RoCE 以及 Ethernet iWARP。 这些技术使用相同的用户态API，但是底层使用不同物理链路。

**（1）InfiniBand**

InfiniBand (IB) is a high-speed, low latency, low CPU overhead, highly efficient and scalable **server and storage interconnect technology**.

IB在主机间传输数据时不需要CPU参与，通过使用I/O channels （up to 16 million per host）传输数据，每个channel都提供虚拟网卡或者HCA的语义能力。

由IBTA组织制定相关标准



**（2）Ethernet RoCE**

RoCE是基于以太网的RDMA解决方案，同样由IBTA组织制定相关标准。

RoCE不需要复杂并且低性能的TCP传输。

It takes advantage of **Priority Flow Control** in Data Center Bridging Ethernet for **lossless connectivity**.



**（3）Ethernet iWARP**

When it comes to the Ethernet solutions, **RoCE has clear performance advantages over iWARP** — both for latency, throughput and CPU overhead. 

【感觉很鸡肋】



### IB关键组件 

**HCA** - Host Channel Adapter:  (类似于以太网中的网卡，但是提供更多的功能)

HCAs provide **address translation mechanism** under the control of the operating system which allows an application to access the HCA directly.

**Switches** 

They implement **flow control of the IB Link Layer to prevent packet dropping**, and to support congestion avoidance and adaptive routing capabilities, and advanced Quality of Service. 

基本都包含 Subnet Manager

**Subnet Manager**

The InfiniBand subnet manager assigns **Local Identifiers (LIDs)** to each port connected to the InfiniBand fabric and develops a routing table based on the assigned LIDs. The IB Subnet Manager is a concept of Software Defined Networking (SDN) which eliminates the interconnect complexity and enables the creation of very large scale compute and storage infrastructures.

**Range Extenders** （不重要）

InfiniBand range extension is accomplished by encapsulating the InfiniBand traffic onto the WAN link and extending sufficient buffer credits to ensure full bandwidth across the WAN.

**Cables**

IB既支持铜缆也支持光缆，在短距离的传输场景下，二者的带宽延迟基本没有差别，长距离传输需要使用光缆。 （光缆的价格远高于铜缆）



## 3. 概念

**QP (Queue Pair)**

a QP is roughly equivalent to a socket. The QP needs to be initialized on both sides of the connection. Communication Manager (CM) can be used to exchange information about the QP prior to actual QP setup.



### 通信操作类型

（1）Send / Receive

（2）RDMA Read / RDMA Write

（3）Atomic Fetch and Add / Atomic Compare and Swap

### 传输模式

![image-20210310153648766](./imgs/image-20210310153648766.png)

* **RC (Reliable Connection)**

  Queue Pair is associated with **only one** other QP.

  Messages transmitted by the send queue of one QP are **reliably** delivered to receive queue of the other QP.

  Packets are delivered **in order**.

  A RC connection is very **similar to a TCP connection**.

* **UC (Unreliable Connection)**

  A Queue Pair is associated with **only one** other QP.

  The connection is not reliable so packets **may be lost**.

  Messages with errors are not retried by the transport, and error handling must be provided by a higher level protocol.

* **UD (Unreliable Datagram)**

  A Queue Pair may transmit and receive single-packet messages to/from **any other UD QP**.

  

  A UD connection is very **similar to a UDP connection**.





CQ (Completion Queue)

CQE (Completion Queue Entries)



AH (Address Handle)

SRQ (Shared Receive Queue)





**WR (Work Request)**





## 4. 编程

介绍具体RDMA编程前，先以类似于传统socket编程的send/recv为例，了解一下RDMA微观的数据传输过程，如图4所示，主要分为以下几步：

- 创建QP、CQ，建立收发两端QP之间的连接（类似于socket功能）；
- 接收端注册用户内存到网卡RNIC，并发起接收的请求，该请求中就包括了注册到网卡的内存地址和长度；
- 发送端注册用户内存到网卡RNIC，并发起发送的请求，该请求中就包括了注册到网卡的内存地址和长度；
- 发送端网卡执行发送请求，根据发送队列SQ中发送请求的内存地址和长度到用户态内存中直接读取数据发送给接收端；
- 当数据到达接收端时，接收端网卡执行接收请求，根据接收队列RQ中接收请求的内存地址和长度将接收的数据直接写到相应的位置；
- 接收端数据接收完成后产生一个完成通知CQE到完成队列CQ中，程序从完成队列中取出完成通知CQE代表整个传输过程的结束。

上述文字描述过程看上去很简单，和socket编程差不多，但是由于RDMA编程库的封装性不强以及在整个传输过程中涉及大量的参数选择，RDMA编程其实极为复杂。



1. 



前面说RDMA在整个传输过程中涉及大量的参数选择，如图5所示，标注了整个过程涉及的主要参数：

- QP类型：RC、UC、UD（R: reliable, U: unreliable, C: connection, D: datagram），QP的类型需要在建立连接时确定，就像在建立socket通信时，需要确定是TCP还是UDP的连接类型。其中，R、U的区别在于是否是可靠传输，也就是是否返回ack，C、D的区别在于是否是面向连接的，C指的是面向连接的，类似于TCP，在数据传输前，会先建立好连接，也就是QP互相交换相应信息，而D指的是datagram，QP之间连接的信息不是事先确定好的，而是放在数据包的包头，由数据包决定所要发送的具体的接收端的QP是哪个；
- Verb：send/recv、write、read，具体的传输数据的方式，send/recv和socket类似，write指的是将数据从本地直接写到远端内存，不需要远端CPU参与，read指的是将数据从远端直接读到本地，同样不需要远端CPU参与；
- Inline/non-inline：inline在C++里指的是内联，在程序编译时直接用函数代码替换函数调用，节省时间，但针对的是那些执行时间较短的函数。同样，在RDMA里面，inline指的就是将一些小的数据包内联在发送请求中，这样在2.1中RDMA数据传输的第四步，就可以少一次取数据的过程；
- Signal/unsignal：signal/unsignal指的是是否产生CQE，如果使用unsignal，将不会产生CQE，因此也就不需要poll CQE，从而减少CPU使用，提高延时。但是，如果使用了unsignal，必须保证隔一段时间发送一个signal的请求，因为如果没有CQE的产生以及poll，那么这些unsignal的发送请求将一直占用着发送队列，当发送队列满时，将不能再post新的请求；
- Poll策略：poll策略指的是poll CQE的方式，包括busy polling、event-triggered polling，busy polling以高CPU使用代价换取更快的CQE poll速度，event-triggered polling则相应的poll速度慢，但CPU使用代价低；

在了解了RDMA各种参数后，接下来将介绍具体的RDMA编程中，这些参数是如何体现的（在具体参数选择出会用黄色背景标注出）。







### Libibverbs API

基于libibverbs的api，应用程序一般按照如下步骤组织：

1. Get the device list (`struct ibv_device **ibv_get_device_list()`)
2. **Create an Infiniband context** (`struct ibv_context* ibv_open_device()`)
3. Create a protection domain (`struct ibv_pd* ibv_alloc_pd()`)
4. Create a completion queue (`struct ibv_cq* ibv_create_cq()`)
5. **Create a queue pair** (`struct ibv_qp* ibv_create_qp()`)
6. Exchange identifier information to establish connection
7. **Change the queue pair state** (`ibv_modify_qp()`): change the state of the queue pair from RESET to INIT, RTR (Ready to Receive), and finally RTS (Ready to Send) 
8. Register a memory region (`ibv_reg_mr()`)
9. Exchange memory region information to handle operations
10. **Perform data communication**

注意：在一些应用程序或者文档中，第8步注册memory region会被作为初始化的一部分，在第2-5步之间执行；事实上，**there is no problem for lazy registration** and memory regions can dynamically be registered and deregistered at any time before posting a work request

**详细说明**

Step 1 Get the device list

`IB devices`对应数据结构 `struct ibv_device`:

```c
struct ibv_device {
	struct _ibv_device_ops	_ops;
	enum ibv_node_type	node_type;
	enum ibv_transport_type	transport_type;
	/* Name of underlying kernel IB device, eg "mthca0" */
	char			name[IBV_SYSFS_NAME_MAX];
	/* Name of uverbs device, eg "uverbs0" */
	char			dev_name[IBV_SYSFS_NAME_MAX];
	/* Path to infiniband_verbs class device in sysfs */
	char			dev_path[IBV_SYSFS_PATH_MAX];
	/* Path to infiniband class device in sysfs */
	char			ibdev_path[IBV_SYSFS_PATH_MAX];
};
```

应用程序通过API `ibv_get_device_list` 获取IB设备列表：

```c
/**
 * ibv_get_device_list - Get list of IB devices currently available
 * @num_devices: optional.  if non-NULL, set to the number of devices
 * returned in the array.
 *
 * Return a NULL-terminated array of IB devices.  The array can be
 * released with ibv_free_device_list().
 */
struct ibv_device **ibv_get_device_list(int *num_devices);
```



Step 2 Create an Infiniband context

遍历第一步得到的 IB devices list，根据设备名称或者GUID选择对应的设备。应用程序调用 `ibv_open_device` 打开IB设备，返回一个 `ibv_context` 对象。

```c
struct ibv_context {
	struct ibv_device      *device;
	struct ibv_context_ops	ops;
	int			cmd_fd;
	int			async_fd;
	int			num_comp_vectors;
	pthread_mutex_t		mutex;
	void		       *abi_compat;
};
```

```c
/**
 * ibv_open_device - Initialize device for use
 */
struct ibv_context *ibv_open_device(struct ibv_device *device);
```



Step 3 Create a protection domain



```c
struct ibv_pd {
	struct ibv_context     *context;
	uint32_t		handle;
};
```



```c
/**
 * ibv_alloc_pd - Allocate a protection domain
 */
struct ibv_pd *ibv_alloc_pd(struct ibv_context *context);
```





Step 4 Create a completion queue

```c
/**
 * ibv_create_cq - Create a completion queue
 * @context - Context CQ will be attached to
 * @cqe - Minimum number of entries required for CQ
 * @cq_context - Consumer-supplied context returned for completion events
 * @channel - Completion channel where completion events will be queued.
 *     May be NULL if completion events will not be used.
 * @comp_vector - Completion vector used to signal completion events.
 *     Must be >= 0 and < context->num_comp_vectors.
 */
struct ibv_cq *ibv_create_cq(struct ibv_context *context, int cqe,
			     void *cq_context,
			     struct ibv_comp_channel *channel,
			     int comp_vector);
```



```c
struct ibv_cq {
	struct ibv_context     *context;
	struct ibv_comp_channel *channel;
	void		       *cq_context;
	uint32_t		handle;
	int			cqe;

	pthread_mutex_t		mutex;
	pthread_cond_t		cond;
	uint32_t		comp_events_completed;
	uint32_t		async_events_completed;
};
```





Step 5 Create a queue pair

```c
/**
 * ibv_create_qp - Create a queue pair.
 */
struct ibv_qp *ibv_create_qp(struct ibv_pd *pd,
			     struct ibv_qp_init_attr *qp_init_attr);
```





Step 6 

- (3)

  Literally create a protection domain, protecting resources from arbitrary accesses from the remote. Components that can be registered to a protection domain is

  - memory regions (MRs)

  - memory windows (MWs)

  - queue pairs (QPs)

  - shared receive queues (SRQs)

  - address handles (AH) 

    ```
    struct ibv_pd* protection_domain = ibv_alloc_pd(context);
    ```

    

    

- (4)

- 

```c
struct ibv_mr {
	struct ibv_context     *context;
	struct ibv_pd	       *pd;
	void		       *addr;
	size_t			length;
	uint32_t		handle;
	uint32_t		lkey;
	uint32_t		rkey;
};
```



```c
/**
 * ibv_reg_mr - Register a memory region
 */
struct ibv_mr *ibv_reg_mr(struct ibv_pd *pd, void *addr,
			  size_t length, int access);
```









## References

1. RDMA Aware Networks Programming User Manual (Rev 1.7)
2. NVIDIA® MLNX_OFED Documentation (Rev 5.2-1.0.4.0)