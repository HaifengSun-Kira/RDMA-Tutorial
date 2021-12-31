RDMA Program

[TOC]

## 介绍

广义的Verbs API主要由两大部分组成：

### IB_VERBS

接口以ibv_xx（用户态）或者ib_xx（内核态）作为前缀，是最基础的编程接口，使用IB_VERBS就足够编写RDMA应用了。

比如：

- ibv_create_qp() 用于创建QP
- ibv_post_send() 用于下发Send WR
- ibv_poll_cq() 用于从CQ中轮询CQE

### RDMA_CM

以rdma_为前缀，主要分为两个功能：

### CMA（Connection Management Abstraction）

在Socket和Verbs API基础上实现的，用于CM建链并交换信息的一组接口。CM建链是在Socket基础上封装为QP实现，从用户的角度来看，是在通过QP交换之后数据交换所需要的QPN，Key等信息。

比如：

- rdma_listen()用于监听链路上的CM建链请求。
- rdma_connect()用于确认CM连接。

### CM VERBS

RDMA_CM也可以用于数据交换，相当于在verbs API上又封装了一套数据交换接口。

比如：

- rdma_post_read()可以直接下发RDMA READ操作的WR，而不像ibv_post_send()，需要在参数中指定操作类型为READ。
- rdma_post_ud_send()可以直接传入远端QPN，指向远端的AH，本地缓冲区指针等信息触发一次UD SEND操作。

上述接口虽然方便，但是需要配合CMA管理的链路使用，不能配合Verbs API使用。

Verbs API除了IB_VERBS和RDMA_CM之外，还有MAD（Management Datagram）接口等。

**需要注意的是，软件栈中的Verbs API具体实现和IB规范中的描述并不是完全一致的。IB规范迭代较慢，而软件栈几乎每天都有变化，所以编写应用或者驱动程序时，应以软件栈API文档中的描述为准。**

狭义的Verbs API专指以ibv_/ib_为前缀的用户态Verbs接口。

## ibv struct

`struct ibv_pd`

- is used to implement protection domains

`struct ibv_mr`

- is used to implement memroy registration

`struct ibv_cq`

- is used to implement a CQ

`struct ibv_wc`

- CQE
- 在`ibv_poll_cq()`中传入一个`ibv_wc`的数组以及数组长度

![](figs/1.PNG)

`struct ibv_qp`

- queue pair
- is created by `ibv_create_qp()`

`struct ibv_qp_init_attr`

![](figs/2.png)



`struct ibv_qp_attr`

- 感觉不常用，应该是在需要修改QP的属性时用

![](figs/3.png)



`struct ibv_ah`

- Is used to implement address vectors
- An Address Vector is an object that describes the route from the local node to the remote node.
- In every UC/RC QP there is an address vector in the QP context.
- In UD QP the address vector should be defined in every post SR.

`struct ibv_sge`

- implements scatter gather elements
- Data is being gathered/scattered using scatter gather elements, which include:
  - Address: address of the local data buffer that the data will be gathered from or scattered to.
  - Size: the size of the data that will be read from / written to this address.
  -  L_key: the local key of the MR that was registered to this buffer.

![](figs/70.png)



`struct ibv_send_wr`

- is used to implement SR

`struct ibv_recv_wr`

- is used to implement RR

`struct ibv_port_attr`

- contain port attributes

![](figs/4.png)

![](figs/5.png)



## ibv api

使用ibv api编程的总体流程：
![](figs/42.jpeg)

### 设备管理

```c
struct ibv_device **ibv_device_list(int *num_devices)
```

![](figs/31.png)

![](figs/32.png)



```c
void ibv_free_device_list(struct ibv_device **list)
```

![](figs/33.png)



```c
struct ibv_context *ibv_open_device(struct ibv_device *device)
```

![](figs/34.png)



```c
int ibv_close_device(struct ibv_context *context)
```

![](figs/35.png)



```c
int ibv_query_device(struct ibv_context *context, struct ibv_device_attr *device_attr);
```

![](figs/36.png)

![](figs/37.png)

![](figs/38.png)



### pd

```c
struct ibv_pd *ibv_alloc_pd(struct ibv_context *context)
```

![](figs/7.png)



```c
int ibv_dealloc_pd(struct ibv_pd *pd)
```

![](figs/11.png)

### mr

```c
struct ibv_mr *ibv_reg_mr(struct ibv_pd *pd, void *addr, size_t length, enum ibv_access_flags access)
```

![](figs/6.png)



```c
int ibv_dereg_mr(struct ibv_mr *mr)
```

![](figs/16.png)

### cq

```c
struct ibv_cq *ibv_create_cq(struct ibv_context *context, int cqe, void *cq_context, struct ibv_comp_channel *channel, int comp_vector)
```

![](figs/8.png)



```c
int ibv_poll_cq(struct ibv_cq *cq, int num_entris, struct ibv_wc *wc)
```

![](figs/9.png)

![](figs/10.png)

- `ibv_poll_cq()` isn't a blocking function and it will always return immediately.
- 参考https://www.rdmamojo.com/2013/02/15/ibv_poll_cq/



```c
int ibv_destroy_cq(struct ibv_cq *cq)
```

![](figs/12.png)

### ah

```c
struct ibv_ah *ibv_create_ah(struct ibv_pd *pd, struct ibv_ah_attr *attr)
```

![](figs/13.png)

![](figs/14.png)



```c
int ibv_destroy_ah(struct ibv_ah *ah)
```

![](figs/15.png)



### qp

```c
struct ibv_qp *ibv_create_qp(struct ibv_pd *pd, struct ibv_qp_init_attr *qp_init_attr)
```

![](figs/21.png)

![](figs/22.png)



```c
int ibv_modify_qp(struct ibv_qp *pq, struct ibv_qp_attr *attr, enum ibv_qp_attr_mask attr_mask)
```

![](figs/41.png)

![](figs/39.png)

![](figs/40.png)



```c
int ibv_query_qp(struct ibv_qp *qp, struct ibv_qp_attr *attr, enum ibv_qp_attr_mask attr_mask, struct ibv_qp_init_attr *init_attr)
```

![](figs/20.png)



```c
int ibv_destroy_qp(struct ibv_qp *qp)
```

![](figs/23.png)

### post rr/sr

```c
int ibv_post_recv(struct ibv_qp *qp, struct ibv_recv_wr *wr, struct ibv_recv_wr **bad_wr);
```

![](figs/19.png)



```c
int ibv_post_send(struct ibv_qp *qp, struct ibv_send_wr *wr, struct ibv_send_wr **bad_wr);
```

![](figs/17.png)

![](figs/18.png)



## rdma struct

`struct rdma_event_channel`

- An event communication channel



`struct rdma_cm_id`

- is conceptually equivalent to a socket for RDMA communication. The difference is that RDMA communication requires explicitly binding to a specified RDMA device before communication can occur, and most operations are asynchronous is nature.
- Communication events on an `rdma_cm_id` are reported through the associated event channel.
- Users must release the `rdma_cm_id` by calling `rdma_destroy_id`.



`struct rdma_cm_event`

- 在不同的传输模式下会出现不同的event

![](figs/26.png)

![](figs/27.png)

![](figs/28.png)

![](figs/29.png)



`struct rdma_conn_param`

![](figs/24.png)

![](figs/25.png)



`struct rdma_addrinfo`

![](figs/30.png)







## rdma api

### event channel

```c
struct rdma_event_channel *rdma_create_event_channel(void);
```

![](figs/43.png)



```c
void rdma_destroy_event_channel(struct rdma_event_channel *channel);
```

![](figs/44.png)

![](figs/45.png)



### Connection Manager(CM) ID 

```c
int rdma_create_id(struct rdma_event_channel *channel, struct rdma_cm_id ** id, void *context, enum rdma_port_space ps);
```

![](figs/46.png)



```c
int rdma_destroy_id(struct rdma_cm_id *id);
```

![](figs/47.png)



```c
int rdma_getaddrinfo(char *node, char *service, struct rdma_addrinfo *hints, struct rdma_addrinfo **res);
```

![](figs/57.png)



```c
void rdma_freeaddrinfo(struct rdma_addrinfo *res);
```

![](figs/58.png)



```c
int rdma_create_ep(struct rdma_cm_id **id, struct rdma_addrinfo *res, struct ibv_pd *pd, struct ibv_qp_init_attr *qp_init_attr);
```

![](figs/59.png)

- After resolving address information using rdma_getaddrinfo, a  user may use this call to allocate an rdma_cm_id based on the results
- If the rdma_cm_id will be used on the active side of a connection, meaning that res->ai_flag does not have RAI_PASSIVE set, rdma_create_ep will automatically create a QP on the rdma_cm_id if qp_init_attr is not NULL.  The QP will be associated with the specified protection domain, if provided, or a default protection domain if not.  Users should see rdma_create_qp for details on the use of the pd and qp_init_attr parameters.  After calling rdma_create_ep, the returned rdma_cm_id may be connected by calling rdma_connect.  The active side calls rdma_resolve_addr and rdma_resolve_route are not necessary.
- If the rdma_cm_id will be used on the passive side of a connection, indicated by having res->ai_flag RAI_PASSIVE set,  this call will save the provided pd and qp_init_attr parameters. When a new connection request is retrieved by calling rdma_get_request, the rdma_cm_id associated with the new connection will automatically be associated with a QP using the pd and qp_init_attr parameters.  **After calling rdma_create_ep, the returned rdma_cm_id may be placed into a listening state by immediately calling rdma_listen**.  The passive side call rdma_bind_addr is not necessary.  Connection requests may then be retrieved by calling rdma_get_request
- 参考：https://man7.org/linux/man-pages/man3/rdma_create_ep.3.html



```c
int rdma_destroy_ep(struct rdma_cm_id *id);
```

![](figs/60.png)



```c
int rdma_resolve_addr(struct rdma_cm_id *id, struct sockaddr *src_addr, struct sockaddr *dst_addr, int timeout_ms);
```

![](figs/48.png)

![](figs/49.png)



```c
int rdma_bind_addr(struct rdma_cm_id *id, struct sockaddr *addr);
```

![](figs/50.png)



```c
int rdma_listen(struct rdma_cm_id *id, int backlog)
```

![](figs/51.png)



```c
int rdma_connect(struct rdma_cm_id *id, struct rdma_conn_param *conn_param)
```

![](figs/52.png)

![](figs/24.png)

![](figs/25.png)



```c
int rdma_get_request(struct rdma_cm_id *listen, struct rdma_cm_id **id);
```

![](figs/53.png)



```c
int rdma_accept(struct rdma_cm_id *id, struct rdma_conn_param *conn_param)
```

![](figs/54.png)



```c
int rdma_reject(struct rdma_cm_id *id, const void *private_data, uint8_t private_data_len);
```

![](figs/55.png)



```c
int rdma_disconnect(struct rdma_cm_id *id);
```

![](figs/56.png)



```c
int rdma_create_qp(struct rdma_cm_id *id, struct ibv_pd *pd, struct ibv_qp_init_attr *qp_init_attr);
```

![](figs/61.png)



```c
void rdma_destroy_qp(struct rdma_cm_id *id);
```

![](figs/62.png)

### memory register

```c
struct ibv_mr *rdma_reg_msgs(struct rdma_cm_id *id, void *addr, size_t length);
```

![](figs/63.png)

```c
struct ibv_mr *rdma_reg_read(struct rdma_cm_id *id, void *addr, size_t length);
```

- Registers a memory buffer that will be accessed by a remote RDMA read operation. 



```c
struct ibv_mr *rdma_reg_write(struct rdma_cm_id *id, void *addr, size_t length);
```

- Registers a memory buffer that will be accessed by a remote RDMA write operation. 



```c
int rdma_dereg_mr(struct ibv_mr *mr);
```

![](figs/64.png)



### event handling

```c
int rdma_get_cm_event(struct rdma_event_channel *channel, struct rdma_cm_event **event);
```

![](figs/73.png)

![](figs/74.png)

![](figs/75.png)



```c
int rdma_ack_cm_event(struct rdma_cm_event *event);
```

![](figs/76.png)



```c
char *rdma_event_str(enum rdma_cm_event_type event);
```

![](figs/77.png)



### post recv/send



```c
int rdma_post_recv(struct rdma_cm_id *id, void *context, void *addr, size_t length, struct ibv_mr *mr);
```

![](figs/65.png)



```c
int rdma_post_recvv(struct rdma_cm_id *id, void *context, struct ibv_sge *sgl, int nsge);
```

![](figs/66.png)

- 这个是support scatter-gather entries.



```c
int rdma_post_send(struct rdma_cm_id *id, void *context, void *addr, size_t length, struct ibv_mr *mr, int flags);
```

![](figs/67.png)



```c
int rdma_post_sendv(struct rdma_cm_id *id, void *context, struct ibv_sge *sgl, int nsge, int flags)
```



```c
int rdma_post_write(struct rdma_cm_id *id, void *context, void *addr, size_t length, struct ibv_mr *mr, int flags, uint64_t remote_addr, uint32_t rkey);
```

![](figs/68.png)



```c
int rdma_post_writev(struct rdma_cm_id *id, void *context, struct ibv_sge *sgl, int nsge, int flags, uint64_t remote_addr, uint32_t rkey);
```



```c
int rdma_post_read(struct rdma_cm_id *id, void *context, void *addr, size_t length, struct ibv_mr *mr, int flags, uint64_t remote_addr, uint32_t rkey);
```

![](figs/69.png)



```c
int rdma_post_readv(struct rdma_cm_id *id, void *context, struct ibv_sge *sgl, int nsge, int flags, uint64_t remote_addr, uint32_t rkey);
```

### get CQE



```c
int rdma_get_send_comp(struct rdma_cm_id *id, struct ibv_wc *wc);
```

![](figs/71.png)



```c
int rdma_get_recv_comp(struct rdma_cm_id *id, struct ibv_wc *wc);
```

![](figs/72.png)







## 有用的网站

https://www.zhihu.com/column/c_1231181516811390976

https://www.rdmamojo.com/

















