# RDMA 学习笔记

1. [RDMA技术背景及原理](./doc/RDMA技术背景及原理.md)

2. [RDMA关键概念](./doc/RDMA关键概念.md)

3. [Libibverbs API 编程](./doc/Libibverbs_API_编程.md)

   ```
   cd verbs_examples
   mkdir build
   cd build
   cmake ..
   make
   
   # 运行 simple_verbs
   # server 端 (dl24):
   ./simple_verbs -d mlx5_1 
   # cliend 端 (dl25):
   ./simple_verbs 10.0.12.24 -d mlx5_1
   ```

4. [Librdmacm API 编程](./doc/Librdmacm_API_编程.md) [TODO]

5. [RDMA paper list](./doc/paper_list.md) [TODO]

6. [perftest]

  ```
  cd perftest-4.4
  ./autogen.sh
  ./configure
  make clean && make V=1
  # server (dl24)
  ./test_case -d mlx5_1
  ./test_case 10.0.12.24 -d mlx5_1
  ```



链接：

1. [RDMA tutorial](https://github.com/jcxue/RDMA-Tutorial/wiki)
2. [RDMA tutorial](https://github.com/StarryVae/RDMA-tutorial/blob/master/tutorial.md)
3. [RDMA tutorial](https://insujang.github.io/2020-02-09/introduction-to-programming-infiniband/)
4. [Performance Tuning for Mellanox Adapters](https://community.mellanox.com/s/article/performance-tuning-for-mellanox-adapters)
5. [MLNX_OFED下载链接 含文档](https://www.mellanox.com/products/infiniband-drivers/linux/mlnx_ofed) 
6. [JVerbs](https://www.ibm.com/support/knowledgecenter/SSYKE2_7.0.0/com.ibm.java.lnx.70.doc/diag/understanding/rdma_jverbs.html)

7. 一个很好的知乎专栏: https://www.zhihu.com/column/c_1231181516811390976

RDMA在整个传输过程中涉及大量的参数选择，整个过程涉及的主要参数：

- QP类型：RC、UC、UD（R: reliable, U: unreliable, C: connection, D: datagram），QP的类型需要在建立连接时确定，就像在建立socket通信时，需要确定是TCP还是UDP的连接类型。其中，R、U的区别在于是否是可靠传输，也就是是否返回ack，C、D的区别在于是否是面向连接的，C指的是面向连接的，类似于TCP，在数据传输前，会先建立好连接，也就是QP互相交换相应信息，而D指的是datagram，QP之间连接的信息不是事先确定好的，而是放在数据包的包头，由数据包决定所要发送的具体的接收端的QP是哪个；

- Verb：send/recv、write、read，具体的传输数据的方式，send/recv和socket类似，write指的是将数据从本地直接写到远端内存，不需要远端CPU参与，read指的是将数据从远端直接读到本地，同样不需要远端CPU参与；

- Inline/non-inline：inline在C++里指的是内联，在程序编译时直接用函数代码替换函数调用，节省时间，但针对的是那些执行时间较短的函数。同样，在RDMA里面，inline指的就是将一些小的数据包内联在发送请求中，这样在2.1中RDMA数据传输的第四步，就可以少一次取数据的过程；

  To enable inlined message. Two places need to be modified: first, add `max_inline_data` to `ibv_qp_init_attr` when create QP; second, set `send_flags` to `IBV_SEND_INLINE`.

- Signal/unsignal：signal/unsignal指的是是否产生CQE，如果使用unsignal，将不会产生CQE，因此也就不需要poll CQE，从而减少CPU使用，提高延时。但是，如果使用了unsignal，必须保证隔一段时间发送一个signal的请求，因为如果没有CQE的产生以及poll，那么这些unsignal的发送请求将一直占用着发送队列，当发送队列满时，将不能再post新的请求；
- Poll策略：poll策略指的是poll CQE的方式，包括busy polling、event-triggered polling，busy polling以高CPU使用代价换取更快的CQE poll速度，event-triggered polling则相应的poll速度慢，但CPU使用代价低；



## References

1. RDMA Aware Networks Programming User Manual (Rev 1.7)
2. NVIDIA® MLNX_OFED Documentation (Rev 5.2-1.0.4.0)
