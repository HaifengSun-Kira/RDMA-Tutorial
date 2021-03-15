# RDMA 学习笔记

## 1. 背景



## 3. 概念







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

To enable inlined message. Two places need to be modified: first, add `max_inline_data` to `ibv_qp_init_attr` when create QP; second, set `send_flags` to `IBV_SEND_INLINE`.



- Signal/unsignal：signal/unsignal指的是是否产生CQE，如果使用unsignal，将不会产生CQE，因此也就不需要poll CQE，从而减少CPU使用，提高延时。但是，如果使用了unsignal，必须保证隔一段时间发送一个signal的请求，因为如果没有CQE的产生以及poll，那么这些unsignal的发送请求将一直占用着发送队列，当发送队列满时，将不能再post新的请求；
- Poll策略：poll策略指的是poll CQE的方式，包括busy polling、event-triggered polling，busy polling以高CPU使用代价换取更快的CQE poll速度，event-triggered polling则相应的poll速度慢，但CPU使用代价低；

在了解了RDMA各种参数后，接下来将介绍具体的RDMA编程中，这些参数是如何体现的（在具体参数选择出会用黄色背景标注出）。



OSDI'18 进程调度，强化学习

(1)RDMA优化app， (2)优化RDMA (hpcc, thu sigcomm19)









## References

1. RDMA Aware Networks Programming User Manual (Rev 1.7)
2. NVIDIA® MLNX_OFED Documentation (Rev 5.2-1.0.4.0)