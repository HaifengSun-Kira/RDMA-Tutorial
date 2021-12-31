# RDMA 学习笔记

1. [RDMA技术背景及原理](./doc/RDMA技术背景及原理.md)

2. [RDMA关键概念](./doc/RDMA关键概念.md)

3. [RDMA入门slide](./slides)

4. [RDMA编程](./doc/RDMA编程.md)

5. [Libibverbs API 编程](./doc/Libibverbs_API_编程.md)

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

6. [RDMA paper list](./paper/paper_list.md) 

7. [perftest](./perftest-4.4) 

  ```
  cd perftest-4.4
  ./autogen.sh
  ./configure
  make clean && make V=1
  # server (dl24)
  ./test_case -d mlx5_1
  ./test_case 10.0.12.24 -d mlx5_1
  ```

7. [官方文档](./pdf)

   * RDMA Aware Networks Programming User Manual (Rev 1.7)
   * NVIDIA® MLNX_OFED Documentation (Rev 5.2-1.0.4.0)

8. [RDMA样例解释](./doc/RDMA样例解释.md)

   

链接：

1. [RDMA tutorial](https://github.com/jcxue/RDMA-Tutorial/wiki)
2. [RDMA tutorial](https://github.com/StarryVae/RDMA-tutorial/blob/master/tutorial.md)
3. [RDMA tutorial](https://insujang.github.io/2020-02-09/introduction-to-programming-infiniband/)
4. [Performance Tuning for Mellanox Adapters](https://community.mellanox.com/s/article/performance-tuning-for-mellanox-adapters)
5. [MLNX_OFED下载链接 含文档](https://www.mellanox.com/products/infiniband-drivers/linux/mlnx_ofed) 
6. [JVerbs](https://www.ibm.com/support/knowledgecenter/SSYKE2_7.0.0/com.ibm.java.lnx.70.doc/diag/understanding/rdma_jverbs.html)
7. [一个很好的知乎专栏**强烈安利**](https://www.zhihu.com/column/c_1231181516811390976) 
8. [RDMA博客**强烈安利**](http://www.rdmamojo.com/)
