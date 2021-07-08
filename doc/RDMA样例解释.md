# 一些基本知识

**MR** : Memory Registration

(context,addr,length,lkey,rkey)

*   buffer可以是虚拟地址连续的，也可以是物理地址连续的
*   本地的HCA使用lkey访问本地内存，rkey被交给远程的HCA去访问远程的内存

**sge** ：scatter/gather entries 发送数据时用到的数据结构

**WQ,RQ,SQ,CQ**: work queue, recv queue, send queue, completion queue

**WQE,CQE**: work queue element



### 简约流程

**Work Request**：一个WR按如下的流程被处理

WR ---> WQ ---> WQE(=WR) ---> Hardware ---> CQ ---> CQE (Work Completion)

*   WQ分为RQ和SQ

*   (RQ,SQ)总是成对被创建，被称为QP

OSI模型中的下面4层(传输层，网络层，数据链路层，物理层)都在RDMA硬件HCA上完成



### 操作类型	Verb：

* Send/Recv
* Read/Pull
* Write/Push
*   Write立即数

**Send/Receive** : 必须成对使用

1.  A和B都MR
2.  B创建一个WQE (RR)到RQ中，包含一个指针指向存放收到数据的地址；A创建WQE (SR)到SQ中，WQE中指针指向待发送的数据
3.  HCA将A中SQ消费掉，B的HCA收到数据后，消费B的WQE，将数据放到缓冲区中
4.  HCA生成CQE

**Read/Write** : 不会通知远程主机，但有立即数的Write会将立即数通知给远程主机



**传输模式（QP类型）**

*   **RC UC UD** R/U:是否可靠（是否返回ACK），C/D:connection/datagram



## Verbs API

### send/receive样例

<img src="https://img-blog.csdnimg.cn/img_convert/3bba5d9bae953ee8ca3519516cbfdec5.png" alt="3bba5d9bae953ee8ca3519516cbfdec5.png" style="zoom: 67%;" />

### 一些函数说明

#### 设备管理

*   ibv_get_device_list *用户获取可用的RDMA设备列表，会返回一组可用设备的指针*
*   ibv_open_device *打开一个可用的RDMA设备，返回其上下文指针（这个指针会在以后用来对这个设备进行各种操作）*
*   ibv_query_device  *查询一个设备的属性/能力，比如其支持的最大QP，CQ数量等。返回设备的属性结构体指针，以及错误码。*
*   ibv_free_device_list *释放ibv_get_device_list获取的RDMA设备列表*
*   ibv_close_device

#### 资源管理

*   ibv_alloc_pd *申请PD。该函数会返回一个PD的指针。(PD(内存)保护域:https://blog.csdn.net/bandaoyu/article/details/112859981* 
*   ibv_create_cq /*创建CQ。用户传入CQ的最小深度（驱动实际申请的可能比这个值大），然后该函数返回CQ的指针。*/ 
*   ibv_create_qp *创建QP。用户传入PD和一组属性（包括RQ和SQ绑定到的CQ、QP绑定的SRQ、QP的能力、QP类型等），向用户返回QP的指针。*
*   ibv_reg_mr *用户应先申请好一篇buffer获取对应VA, 再调用该接口注册MR。用户传入要注册的内存的起始地址和长度，以及这个MR将要从属的PD和它的访问权限（本地读/写，远端读/写等），返回一个MR的指针给用户。*
*   ibv_modiy_qp *修改QP。用户传入QP的指针，以及表示要修改的属性的掩码和要修改值。修改的内容可以是QP状态、对端QPN(QP的序号)、QP能力、端口号和重传次数等等。如果失败，该函数会返回错误码。Modify QP最重要的作用是让QP在不同的状态间迁移，完成RST-->INIT-->RTR-->RTS的状态机转移后才具备下发Send WR的能力。也可用来将QP切换到ERROR状态。*
*   ibv_destroy_qp *销毁QP。即销毁QP相关的软件资源。其他的资源也都有类似的销毁接口。*

#### 中断处理

*   ibv_get_async_event *从事件队列中获取一个异步事件，返回异步事件的信息（事件来源，事件类型等）以及错误码。

### 主要代码框架

1.  初始化

```c
dev_list = ibv_get_device_list(NULL);	// 获取设备列表

for (i = 0; dev_list[i]; ++i)
    if (!strcmp(ibv_get_device_name(dev_list[i]), ib_devname))	// 选择设备
    break;
	ib_dev = dev_list[i];
...
    
ctx = pp_init_ctx(ib_dev, size, rx_depth, ib_port, use_event);
{
    ctx = calloc(1, sizeof *ctx); ...
    ctx->buf = memalign(page_size, size); ...
    ctx->context = ibv_open_device(ib_dev); ...
        // context主要是关于device的一些信息
    ctx->pd = ibv_alloc_pd(ctx->context);
    	// pd:client可以控制哪些remote computers可以访问它的内存区域
    ibv_reg_mr(ctx->pd, ctx->buf, size, access_flags);
    	// access_flags = IBV_ACCESS_LOCAL_WRITE | ... 规定了mr能进行的op的权限
    ctx->cq_s.cq = ibv_create_cq(ctx->context, rx_depth + 1 /* |SQ|+|RQ| */, NULL,
					     ctx->channel, 0); ...
    	struct ibv_qp_init_attr init_attr = {
			.send_cq = ctx->cq,
			.recv_cq = ctx->cq, 
			.cap     = {
				.max_send_wr  = 1, //SQ能容纳的WR数
				.max_recv_wr  = rx_depth,	//RQ能容纳的WR数
				.max_send_sge = 1, //每个WR能容纳的SGE数
				.max_recv_sge = 1
			},
			.qp_type = IBV_QPT_RC // 规定QP类型：reliable connection
		};
    ctx->qp = ibv_create_qp(ctx->pd, &init_attr);
    struct ibv_qp_attr attr={
        ...
        .qp_access_flags = 0 // 在read/write时要修改这里
    }
    if (ibv_modify_qp(ctx->qp, &attr,
				  IBV_QP_STATE              |
				  IBV_QP_PKEY_INDEX         |
				  IBV_QP_PORT               |
				  IBV_QP_ACCESS_FLAGS)) {	// important
			fprintf(stderr, "Failed to modify QP to INIT\n");
			goto clean_qp;
		}	//将QP移动到INIT状态
}

```

2.  先将一个list的WR放到QP上

```c
routs = pp_post_recv(ctx, ctx->rx_depth);
	// Receive buffers must be posted before any sends.
pp_post_recv(ctx,n)
{
    struct ibv_sge list = {	
		.addr	= (uintptr_t) ctx->buf,
		.length = ctx->size,
		.lkey	= ctx->mr->lkey
	};
	struct ibv_recv_wr wr = {
		.wr_id	    = PINGPONG_RECV_WRID,
		.sg_list    = &list,
		.num_sge    = 1,
	};
	struct ibv_recv_wr *bad_wr;
	int i;
	for (i = 0; i < n; ++i)
		if (ibv_post_recv(ctx->qp, &wr, &bad_wr))
			break;
}



```

3.  my_dest <----> rem_dest：使用**TCP socket**互相交换信息。交换的内容：

*   LID : Local Identifier : ctx->portinfo.lid
*   QPN : Queue Pair Number : ctx->qp->qp_num
*   PSN : Packet Sequence Number : lrand48() & 0xffffff
*   GID : 识别一个Port

```c
// 根据是client/server来获取对面的信息
rem_dest = pp_client_exch_dest(servername, port, &my_dest);
rem_dest = pp_server_exch_dest(ctx, ib_port, mtu, port, sl,
								&my_dest, gidx);
```





4.  **连接QP**：一开始QP处于RESET状态，需将其转换成INIT状态；然后将其转换成RTR(Ready to receive)，再转换成RTS(ready to send)状态

```c
pp_connect_ctx(ctx, ib_port, my_dest->psn, mtu, sl, rem_dest,
								sgid_idx)
```

5.  在client端发送一个send，然后轮询cq

```c
do {
    ne = ibv_poll_cq(ctx->cq, 2, wc);
} while (ne < 1);

for (i = 0; i < ne; ++i) {
    ret = parse_single_wc(ctx, &scnt, &rcnt, &routs,
                          iters,
                          wc[i].wr_id,
                          wc[i].status,
                          0, &ts);
    if (ret) {
        fprintf(stderr, "parse WC failed %d\n", ne);
        return 1;
    }
}
```

在处理每个CQE的时候进行pingpong

**注意**：**Flow Control** 在send之前receiver必须有足够的posted receives；CQ不能overflow

```c
parse_single_wc()
{
    switch ((int)wr_id) {
	case PINGPONG_SEND_WRID:
		++(*scnt);
		break;
	case PINGPONG_RECV_WRID:
        if (--(*routs) <= 1) {
			// 再放入post 500个recv，保证有足够的recv
		}
    ...
    }
    
    ctx->pending &= ~(int)wr_id;
	if (*scnt < iters && !ctx->pending) { //如果需要更多的send且没有正在等待完成的send/receive
		if (pp_post_send(ctx)) {
			fprintf(stderr, "Couldn't post send\n");
			return 1;
		}
		ctx->pending = PINGPONG_RECV_WRID |
			PINGPONG_SEND_WRID;
	}
}
```

所以实际效果：

| client                   |      | sever                   |
| ------------------------ | ---- | ----------------------- |
| RQ:RECV； SQ: ; CQ:      |      | RQ: RECV; SQ: ; CQ:     |
| RQ: RECV; SQ: SEND; CQ:  | -->  | RQ: RECV; SQ: ; CQ:     |
| RQ: RECV; SQ: ; CQ: SEND |      | RQ: ; SQ: ; CQ: RECV    |
| RQ: RECV; SQ: ; CQ:      | <--- | RQ: RECV; SQ: SEND; CQ: |
|                          |      |                         |

>   附一篇关于rc_pingpong.c的很好的解释https://arxiv.org/pdf/1105.1827.pdf

### read/write

代码的主要区别在于

```c
// 1, 修改mr的access_flags使其支持remote read/write
ctx->mr = ibv_reg_mr(ctx->pd, ctx->buf, size, IBV_ACCESS_LOCAL_WRITE | 
                                          IBV_ACCESS_REMOTE_READ |
                                          IBV_ACCESS_REMOTE_WRITE);
// 2. 修改qp的access_flags
struct ibv_qp_attr attr1 = {
            .qp_state        = IBV_QPS_INIT,
            .pkey_index      = 0,
            .port_num        = port,
            .qp_access_flags = IBV_ACCESS_LOCAL_WRITE |
                               IBV_ACCESS_REMOTE_READ |
                               IBV_ACCESS_REMOTE_WRITE,
        };
// 记得在下面开启IBV_QP_ACCESS_FLAGS
ibv_modify_qp(ctx->qp2, &attr2,
                  IBV_QP_STATE              |
                  IBV_QP_PKEY_INDEX         |
                  IBV_QP_PORT               |
                  IBV_QP_ACCESS_FLAGS)
// 3. 在socket发送消息的时候发送mr的.addr和.rkey
// 4. post_read/post_write
static int pp_post_read(struct pingpong_context *ctx)
{
    struct ibv_sge list = {
        .addr	= (uintptr_t) ctx->buf2,
        .length = ctx->size,
        .lkey	= ctx->mr2->lkey,
    };
    struct ibv_send_wr wr = {
        .wr_id	    = PP_READ_WRID,
        .sg_list    = &list,
        .num_sge    = 1,
        .opcode     = IBV_WR_RDMA_READ,
        .send_flags = ctx->send_flags,
        .wr.rdma.remote_addr = (uintptr_t) ctx->addr2,
        .wr.rdma.rkey = ctx->rkey2,
    };
    struct ibv_send_wr *bad_wr;
    return ibv_post_send(ctx->qp2, &wr, &bad_wr);
}
   static int pp_post_write(struct pingpong_context *ctx)
{
    struct ibv_sge list = {
        .addr	= (uintptr_t) ctx->buf,
        .length = ctx->size,
        .lkey	= ctx->mr1->lkey,
    };
    struct ibv_send_wr wr = {
        .wr_id	    = PP_WRITE_WRID,	
        .sg_list    = &list,
        .num_sge    = 1,
        .opcode     = IBV_WR_RDMA_WRITE,
        .send_flags = ctx->send_flags,
        .wr.rdma.remote_addr = (uintptr_t) ctx->addr,
        .wr.rdma.rkey = ctx->rkey,
    };
    struct ibv_send_wr *bad_wr;
    return ibv_post_send(ctx->qp1, &wr, &bad_wr);
}

```

---

附：自己改写的read/write代码：

*   client和server都有2个buffer，client的buf1向server的buf1中写，client的buf2向server的buf2中读

*   将其放入verbs_examples中，与其他样例一同编译即可

```c
/*
 * Copyright (c) 2005 Topspin Communications.  All rights reserved.
 *
 * This software is available to you under a choice of one of two
 * licenses.  You may choose to be licensed under the terms of the GNU
 * General Public License (GPL) Version 2, available from the file
 * COPYING in the main directory of this source tree, or the
 * OpenIB.org BSD license below:
 *
 *     Redistribution and use in source and binary forms, with or
 *     without modification, are permitted provided that the following
 *     conditions are met:
 *
 *      - Redistributions of source code must retain the above
 *        copyright notice, this list of conditions and the following
 *        disclaimer.
 *
 *      - Redistributions in binary form must reproduce the above
 *        copyright notice, this list of conditions and the following
 *        disclaimer in the documentation and/or other materials
 *        provided with the distribution.
 *
 * THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND,
 * EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
 * MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND
 * NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS
 * BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN
 * ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN
 * CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
 * SOFTWARE.
 */
#define _GNU_SOURCE
#include "config.h"

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <sys/time.h>
#include <netdb.h>
#include <malloc.h>
#include <getopt.h>
#include <arpa/inet.h>
#include <time.h>
#include <inttypes.h>

#include "pingpong.h"

#include "ccan/minmax.h"

enum {
    PINGPONG_RECV_WRID = 1,
    PINGPONG_SEND_WRID = 2,
    PP_WRITE_WRID = 4,
    PP_READ_WRID  = 8,
};

static int page_size;
static int use_odp;
static int implicit_odp;
static int prefetch_mr;
static int use_ts;
static int validate_buf;
static int use_dm;
static int use_new_send;

struct pingpong_context {
    struct ibv_context	*context;
    struct ibv_comp_channel *channel;
    struct ibv_pd		*pd;
    struct ibv_mr		*mr1;
    struct ibv_mr		*mr2;
    struct ibv_dm		*dm;
    union {
        struct ibv_cq		*cq;
        struct ibv_cq_ex	*cq_ex;
    } cq_s;
    struct ibv_qp		*qp1;
    struct ibv_qp		*qp2;
    struct ibv_qp_ex	*qpx;
    char			*buf1;
    char			*buf2;
    int			 size;
    int			 send_flags;
    int			 rx_depth;
    int			 pending;
    void            *addr1;
    void            *addr2;
    uint32_t        rkey1;
    uint32_t        rkey2;
    struct ibv_port_attr     portinfo;
    uint64_t		 completion_timestamp_mask;
};

static struct ibv_cq *pp_cq(struct pingpong_context *ctx)
{
    return use_ts ? ibv_cq_ex_to_cq(ctx->cq_s.cq_ex) :
        ctx->cq_s.cq;
}

struct pingpong_dest {
    int lid;
    int qpn1;
    int qpn2;
    int psn1;
    int psn2;
    void *addr1;
    void *addr2;
    uint32_t rkey1;
    uint32_t rkey2;
    union ibv_gid gid;
};

static int pp_connect_ctx(struct pingpong_context *ctx, int port, 
              int my_psn1, int my_psn2,
              enum ibv_mtu mtu, int sl,
              struct pingpong_dest *dest, int sgid_idx)
{
    struct ibv_qp_attr attr1 = {
        .qp_state		= IBV_QPS_RTR,
        .path_mtu		= mtu,
        .dest_qp_num		= dest->qpn1,
        .rq_psn			= dest->psn1,
        .max_dest_rd_atomic	= 1,
        .min_rnr_timer		= 12,
        .ah_attr		= {
            .is_global	= 0,
            .dlid		= dest->lid,
            .sl		= sl,
            .src_path_bits	= 0,
            .port_num	= port
        }
    };

    struct ibv_qp_attr attr2 = {
        .qp_state		= IBV_QPS_RTR,
        .path_mtu		= mtu,
        .dest_qp_num		= dest->qpn2,
        .rq_psn			= dest->psn2,
        .max_dest_rd_atomic	= 1,
        .min_rnr_timer		= 12,
        .ah_attr		= {
            .is_global	= 0,
            .dlid		= dest->lid,
            .sl		= sl,
            .src_path_bits	= 0,
            .port_num	= port
        }
    };

    if (dest->gid.global.interface_id) {
        attr1.ah_attr.is_global = 1;
        attr1.ah_attr.grh.hop_limit = 1;
        attr1.ah_attr.grh.dgid = dest->gid;
        attr1.ah_attr.grh.sgid_index = sgid_idx;

        attr2.ah_attr.is_global = 1;
        attr2.ah_attr.grh.hop_limit = 1;
        attr2.ah_attr.grh.dgid = dest->gid;
        attr2.ah_attr.grh.sgid_index = sgid_idx;
    }
    if (ibv_modify_qp(ctx->qp1, &attr1,
              IBV_QP_STATE              |
              IBV_QP_AV                 |
              IBV_QP_PATH_MTU           |
              IBV_QP_DEST_QPN           |
              IBV_QP_RQ_PSN             |
              IBV_QP_MAX_DEST_RD_ATOMIC |
              IBV_QP_MIN_RNR_TIMER) ||
        ibv_modify_qp(ctx->qp2, &attr2,
              IBV_QP_STATE              |
              IBV_QP_AV                 |
              IBV_QP_PATH_MTU           |
              IBV_QP_DEST_QPN           |
              IBV_QP_RQ_PSN             |
              IBV_QP_MAX_DEST_RD_ATOMIC |
              IBV_QP_MIN_RNR_TIMER)) {
        fprintf(stderr, "Failed to modify QP to RTR\n");
        return 1;
    }

    attr1.qp_state	    = IBV_QPS_RTS;
    attr1.timeout	    = 14;
    attr1.retry_cnt	    = 7;
    attr1.rnr_retry	    = 7;
    attr1.sq_psn	    = my_psn1;
    attr1.max_rd_atomic  = 1;

    attr2.qp_state	    = IBV_QPS_RTS;
    attr2.timeout	    = 14;
    attr2.retry_cnt	    = 7;
    attr2.rnr_retry	    = 7;
    attr2.sq_psn	    = my_psn2;
    attr2.max_rd_atomic  = 1;
    if (ibv_modify_qp(ctx->qp1, &attr1,
              IBV_QP_STATE              |
              IBV_QP_TIMEOUT            |
              IBV_QP_RETRY_CNT          |
              IBV_QP_RNR_RETRY          |
              IBV_QP_SQ_PSN             |
              IBV_QP_MAX_QP_RD_ATOMIC) ||
        ibv_modify_qp(ctx->qp2, &attr2,
              IBV_QP_STATE              |
              IBV_QP_TIMEOUT            |
              IBV_QP_RETRY_CNT          |
              IBV_QP_RNR_RETRY          |
              IBV_QP_SQ_PSN             |
              IBV_QP_MAX_QP_RD_ATOMIC)) {
        fprintf(stderr, "Failed to modify QP to RTS\n");
        return 1;
    }

    return 0;
}

static struct pingpong_dest *pp_client_exch_dest(const char *servername, int port,
                         const struct pingpong_dest *my_dest)
{
    struct addrinfo *res, *t;
    struct addrinfo hints = {
        .ai_family   = AF_UNSPEC,
        .ai_socktype = SOCK_STREAM
    };
    char *service;
    char msg[sizeof "0000:000000:000000:000000:000000:00000000000000:000000:00000000000000:000000:00000000000000000000000000000000"];
    int n;
    int sockfd = -1;
    struct pingpong_dest *rem_dest = NULL;
    char gid[33];

    if (asprintf(&service, "%d", port) < 0)
        return NULL;

    n = getaddrinfo(servername, service, &hints, &res);

    if (n < 0) {
        fprintf(stderr, "%s for %s:%d\n", gai_strerror(n), servername, port);
        free(service);
        return NULL;
    }

    for (t = res; t; t = t->ai_next) {
        sockfd = socket(t->ai_family, t->ai_socktype, t->ai_protocol);
        if (sockfd >= 0) {
            if (!connect(sockfd, t->ai_addr, t->ai_addrlen))
                break;
            close(sockfd);
            sockfd = -1;
        }
    }

    freeaddrinfo(res);
    free(service);

    if (sockfd < 0) {
        fprintf(stderr, "Couldn't connect to %s:%d\n", servername, port);
        return NULL;
    }

    gid_to_wire_gid(&my_dest->gid, gid);
    sprintf(msg, "%04x:%06x:%06x:%06x:%06x:%014p:%06x:%014p:%06x:%s", my_dest->lid, 
                            my_dest->qpn1, my_dest->qpn2,
                            my_dest->psn1, my_dest->psn2,
                            (uintptr_t) my_dest->addr1, my_dest->rkey1,
                            (uintptr_t) my_dest->addr2, my_dest->rkey2, gid);
    
    printf("%s\n",msg);

    if (write(sockfd, msg, sizeof msg) != sizeof msg) {
        fprintf(stderr, "Couldn't send local address\n");
        goto out;
    }

    if (read(sockfd, msg, sizeof msg) != sizeof msg ||
        write(sockfd, "done", sizeof "done") != sizeof "done") {
        perror("client read/write");
        fprintf(stderr, "Couldn't read/write remote address\n");
        goto out;
    }

    rem_dest = malloc(sizeof *rem_dest);
    if (!rem_dest)
        goto out;

    sscanf(msg, "%x:%x:%x:%x:%x:%p:%x:%p:%x:%s", &rem_dest->lid, 
                            &rem_dest->qpn1, &rem_dest->qpn2,
                            &rem_dest->psn1, &rem_dest->psn2, 
                            &rem_dest->addr1, &rem_dest->rkey1,
                            &rem_dest->addr2, &rem_dest->rkey2, gid);
    wire_gid_to_gid(gid, &rem_dest->gid);

out:
    close(sockfd);
    return rem_dest;
}

static struct pingpong_dest *pp_server_exch_dest(struct pingpong_context *ctx,
                         int ib_port, enum ibv_mtu mtu,
                         int port, int sl,
                         const struct pingpong_dest *my_dest,
                         int sgid_idx)
{
    struct addrinfo *res, *t;
    struct addrinfo hints = {
        .ai_flags    = AI_PASSIVE,
        .ai_family   = AF_UNSPEC,
        .ai_socktype = SOCK_STREAM
    };
    char *service;
    char msg[sizeof "0000:000000:000000:000000:000000:00000000000000:000000:00000000000000:000000:00000000000000000000000000000000"];
    int n;
    int sockfd = -1, connfd;
    struct pingpong_dest *rem_dest = NULL;
    char gid[33];

    if (asprintf(&service, "%d", port) < 0)
        return NULL;

    n = getaddrinfo(NULL, service, &hints, &res);

    if (n < 0) {
        fprintf(stderr, "%s for port %d\n", gai_strerror(n), port);
        free(service);
        return NULL;
    }

    for (t = res; t; t = t->ai_next) {
        sockfd = socket(t->ai_family, t->ai_socktype, t->ai_protocol);
        if (sockfd >= 0) {
            n = 1;

            setsockopt(sockfd, SOL_SOCKET, SO_REUSEADDR, &n, sizeof n);

            if (!bind(sockfd, t->ai_addr, t->ai_addrlen))
                break;
            close(sockfd);
            sockfd = -1;
        }
    }

    freeaddrinfo(res);
    free(service);

    if (sockfd < 0) {
        fprintf(stderr, "Couldn't listen to port %d\n", port);
        return NULL;
    }

    listen(sockfd, 1);
    connfd = accept(sockfd, NULL, NULL);
    close(sockfd);
    if (connfd < 0) {
        fprintf(stderr, "accept() failed\n");
        return NULL;
    }

    n = read(connfd, msg, sizeof msg);
    if (n != sizeof msg) {
        perror("server read");
        fprintf(stderr, "%d/%d: Couldn't read remote address\n", n, (int) sizeof msg);
        goto out;
    }

    rem_dest = malloc(sizeof *rem_dest);
    if (!rem_dest)
        goto out;

    sscanf(msg, "%x:%x:%x:%x:%x:%p:%x:%p:%x:%s", &rem_dest->lid, 
                            &rem_dest->qpn1, &rem_dest->qpn2,
                            &rem_dest->psn1, &rem_dest->psn2, 
                            &rem_dest->addr1, &rem_dest->rkey1,
                            &rem_dest->addr2, &rem_dest->rkey2, gid);
    wire_gid_to_gid(gid, &rem_dest->gid);

    if (pp_connect_ctx(ctx, ib_port, my_dest->psn1, my_dest->psn2, mtu, sl, rem_dest,
                                sgid_idx)) {
        fprintf(stderr, "Couldn't connect to remote QP\n");
        free(rem_dest);
        rem_dest = NULL;
        goto out;
    }


    gid_to_wire_gid(&my_dest->gid, gid);
    sprintf(msg, "%04x:%06x:%06x:%06x:%06x:%014p:%06x:%014p:%06x:%s", my_dest->lid, 
                            my_dest->qpn1, my_dest->qpn2,
                            my_dest->psn1, my_dest->psn2,
                            (uintptr_t) my_dest->addr1, my_dest->rkey1,
                            (uintptr_t) my_dest->addr2, my_dest->rkey2, gid);

    printf("%s\n",msg);

    printf("write complete\n");
    if (write(connfd, msg, sizeof msg) != sizeof msg ||
        read(connfd, msg, sizeof msg) != sizeof "done") {
        fprintf(stderr, "Couldn't send/recv local address\n");
        free(rem_dest);
        rem_dest = NULL;
        goto out;
    }

    printf("ready to close\n");

out:
    close(connfd);
    return rem_dest;
}

static struct pingpong_context *pp_init_ctx(struct ibv_device *ib_dev, int size,
                        int rx_depth, int port,
                        int use_event)
{
    struct pingpong_context *ctx;
    int access_flags = IBV_ACCESS_LOCAL_WRITE;

    char *ptr;
    int i;

    ctx = calloc(1, sizeof *ctx);
    if (!ctx)
        return NULL;

    ctx->size       = size;
    ctx->send_flags = IBV_SEND_SIGNALED;
    ctx->rx_depth   = rx_depth;

    ctx->buf1 = memalign(page_size, size);
    if (!ctx->buf1) {
        fprintf(stderr, "Couldn't allocate work buf1.\n");
        goto clean_ctx;
    }

    ctx->buf2 = memalign(page_size, size);
    if (!ctx->buf2) {
        fprintf(stderr, "Couldn't allocate work buf2.\n");
        goto clean_ctx;
    }

    /* FIXME memset(ctx->buf, 0, size); */
    // memset(ctx->buf, 0x7b, size);
    for (ptr = ctx->buf1, i = 0; i < size; i++, ptr++)
    {
        *ptr = (char)rand();
    }

    ctx->context = ibv_open_device(ib_dev);
    if (!ctx->context) {
        fprintf(stderr, "Couldn't get context for %s\n",
            ibv_get_device_name(ib_dev));
        goto clean_buffer;
    }

    ctx->channel = NULL;

    ctx->pd = ibv_alloc_pd(ctx->context);
    if (!ctx->pd) {
        fprintf(stderr, "Couldn't allocate PD\n");
        goto clean_comp_channel;
    }


    if (use_odp || use_dm || use_ts)
    {
        fprintf(stderr, "Not implemented yet\n");
        goto clean_pd;
    }
    ctx->mr1 = ibv_reg_mr(ctx->pd, ctx->buf1, size, access_flags | 
                                          IBV_ACCESS_REMOTE_READ |
                                          IBV_ACCESS_REMOTE_WRITE);
    ctx->mr2 = ibv_reg_mr(ctx->pd, ctx->buf2, size, access_flags | 
                                          IBV_ACCESS_REMOTE_READ |
                                          IBV_ACCESS_REMOTE_WRITE);


    if (!ctx->mr1 || !ctx->mr2) {
        fprintf(stderr, "Couldn't register MR\n");
        goto clean_dm;
    }

    
    ctx->cq_s.cq = ibv_create_cq(ctx->context, rx_depth*2 + 4, NULL,
                        ctx->channel, 0);

    if (!pp_cq(ctx)) {
        fprintf(stderr, "Couldn't create CQ\n");
        goto clean_mr;
    }

    {
        struct ibv_qp_attr attr;
        struct ibv_qp_init_attr init_attr = {
            .send_cq = pp_cq(ctx),
            .recv_cq = pp_cq(ctx),
            .cap     = {
                .max_send_wr  = 2,
                .max_recv_wr  = rx_depth,
                .max_send_sge = 1,
                .max_recv_sge = 1
            },
            .qp_type = IBV_QPT_RC
        };

        ctx->qp1 = ibv_create_qp(ctx->pd, &init_attr);
        ctx->qp2 = ibv_create_qp(ctx->pd, &init_attr);

        if (!ctx->qp1 || !ctx->qp2)  {
            fprintf(stderr, "Couldn't create QP\n");
            goto clean_cq;
        }

        ibv_query_qp(ctx->qp1, &attr, IBV_QP_CAP, &init_attr);
        if (init_attr.cap.max_inline_data >= size && !use_dm)
            ctx->send_flags |= IBV_SEND_INLINE;

        ibv_query_qp(ctx->qp2, &attr, IBV_QP_CAP, &init_attr);
        if (init_attr.cap.max_inline_data >= size && !use_dm)
            ctx->send_flags |= IBV_SEND_INLINE;
    }

    {
        struct ibv_qp_attr attr1 = {
            .qp_state        = IBV_QPS_INIT,
            .pkey_index      = 0,
            .port_num        = port,
            .qp_access_flags = IBV_ACCESS_LOCAL_WRITE |
                               IBV_ACCESS_REMOTE_READ |
                               IBV_ACCESS_REMOTE_WRITE,
        };
        struct ibv_qp_attr attr2 = {
            .qp_state        = IBV_QPS_INIT,
            .pkey_index      = 0,
            .port_num        = port,
            .qp_access_flags = IBV_ACCESS_LOCAL_WRITE |
                               IBV_ACCESS_REMOTE_READ |
                               IBV_ACCESS_REMOTE_WRITE,
        };

        if (ibv_modify_qp(ctx->qp1, &attr1,
                  IBV_QP_STATE              |
                  IBV_QP_PKEY_INDEX         |
                  IBV_QP_PORT               |
                  IBV_QP_ACCESS_FLAGS) ||
            ibv_modify_qp(ctx->qp2, &attr2,
                  IBV_QP_STATE              |
                  IBV_QP_PKEY_INDEX         |
                  IBV_QP_PORT               |
                  IBV_QP_ACCESS_FLAGS)) {
            fprintf(stderr, "Failed to modify QP to INIT\n");
            goto clean_qp;
        }
    }

    return ctx;

clean_qp:
    ibv_destroy_qp(ctx->qp1);
    ibv_destroy_qp(ctx->qp2);

clean_cq:
    ibv_destroy_cq(pp_cq(ctx));

clean_mr:
    ibv_dereg_mr(ctx->mr1);
    ibv_dereg_mr(ctx->mr2);

clean_dm:
    if (ctx->dm)
        ibv_free_dm(ctx->dm);

clean_pd:
    ibv_dealloc_pd(ctx->pd);

clean_comp_channel:
    if (ctx->channel)
        ibv_destroy_comp_channel(ctx->channel);

clean_device:
    ibv_close_device(ctx->context);

clean_buffer:
    free(ctx->buf1);
    free(ctx->buf2);

clean_ctx:
    free(ctx);

    return NULL;
}

static int pp_close_ctx(struct pingpong_context *ctx)
{
    if (ibv_destroy_qp(ctx->qp1) || ibv_destroy_qp(ctx->qp2)) {
        fprintf(stderr, "Couldn't destroy QP\n");
        return 1;
    }

    if (ibv_destroy_cq(pp_cq(ctx))) {
        fprintf(stderr, "Couldn't destroy CQ\n");
        return 1;
    }

    if (ibv_dereg_mr(ctx->mr1) || ibv_dereg_mr(ctx->mr2)) {
        fprintf(stderr, "Couldn't deregister MR\n");
        return 1;
    }

    if (ctx->dm) {
        if (ibv_free_dm(ctx->dm)) {
            fprintf(stderr, "Couldn't free DM\n");
            return 1;
        }
    }

    if (ibv_dealloc_pd(ctx->pd)) {
        fprintf(stderr, "Couldn't deallocate PD\n");
        return 1;
    }

    if (ctx->channel) {
        if (ibv_destroy_comp_channel(ctx->channel)) {
            fprintf(stderr, "Couldn't destroy completion channel\n");
            return 1;
        }
    }

    if (ibv_close_device(ctx->context)) {
        fprintf(stderr, "Couldn't release context\n");
        return 1;
    }

    free(ctx->buf1);
    free(ctx->buf2);
    free(ctx);

    return 0;
}

static int pp_post_recv(struct pingpong_context *ctx, int n)
{
    struct ibv_sge list = {
        .addr	= use_dm ? 0 : (uintptr_t) ctx->buf1,
        .length = ctx->size,
        .lkey	= ctx->mr1->lkey,
    };
    struct ibv_recv_wr wr = {
        .wr_id	    = PINGPONG_RECV_WRID,
        .sg_list    = &list,
        .num_sge    = 1,
    };
    struct ibv_recv_wr *bad_wr;
    int i;

    for (i = 0; i < n; ++i)
        if (ibv_post_recv(ctx->qp1, &wr, &bad_wr))
            break;

    return i;
}

static int pp_post_read(struct pingpong_context *ctx)
{
    struct ibv_sge list = {
        .addr	= (uintptr_t) ctx->buf2,
        .length = ctx->size,
        .lkey	= ctx->mr2->lkey,
    };
    struct ibv_send_wr wr = {
        .wr_id	    = PP_READ_WRID,
        .sg_list    = &list,
        .num_sge    = 1,
        .opcode     = IBV_WR_RDMA_READ,
        .send_flags = ctx->send_flags,
        .wr.rdma.remote_addr = (uintptr_t) ctx->addr2,
        .wr.rdma.rkey = ctx->rkey2,
    };
    struct ibv_send_wr *bad_wr;
    return ibv_post_send(ctx->qp2, &wr, &bad_wr);
}

static int pp_post_write(struct pingpong_context *ctx)
{
    struct ibv_sge list = {
        .addr	= (uintptr_t) ctx->buf1,
        .length = ctx->size,
        .lkey	= ctx->mr1->lkey,
    };
    struct ibv_send_wr wr = {
        .wr_id	    = PP_WRITE_WRID,
        .sg_list    = &list,
        .num_sge    = 1,
        .opcode     = IBV_WR_RDMA_WRITE,
        .send_flags = ctx->send_flags,
        .wr.rdma.remote_addr = (uintptr_t) ctx->addr1,
        .wr.rdma.rkey = ctx->rkey1,
    };
    struct ibv_send_wr *bad_wr;
    return ibv_post_send(ctx->qp1, &wr, &bad_wr);
}

struct ts_params {
    uint64_t		 comp_recv_max_time_delta;
    uint64_t		 comp_recv_min_time_delta;
    uint64_t		 comp_recv_total_time_delta;
    uint64_t		 comp_recv_prev_time;
    int			 last_comp_with_ts;
    unsigned int		 comp_with_time_iters;
};

static inline int parse_single_wc(struct pingpong_context *ctx, int *scnt,
                  int *rcnt, int *routs, int iters,
                  uint64_t wr_id, enum ibv_wc_status status,
                  uint64_t completion_timestamp,
                  struct ts_params *ts)
{
    if (status != IBV_WC_SUCCESS) {
        fprintf(stderr, "Failed status %s (%d) for wr_id %d\n",
            ibv_wc_status_str(status),
            status, (int)wr_id);
        return 1;
    }

    switch ((int)wr_id) {
    case PP_READ_WRID:
        ++(*rcnt);
        printf("READ Complete\n");
        if (*rcnt < iters ) {
            int ret;
            if (ret = pp_post_read(ctx)) {
                fprintf(stderr, "Couldn't post read\n");
                printf("ret %d, EINVAL %d, ENOMEM %d, EFAULT %d\n", ret, EINVAL, ENOMEM, EFAULT);
                printf("scnt %d, rcnt %d\n",scnt,rcnt);
                fprintf(stderr, "Couldn't post write\n");
                return 1;
            }
        }
        break;

    case PP_WRITE_WRID:
        ++(*scnt);
        printf("WRITE Complete\n");
        if (*scnt < iters ) {
            int ret;
            if (ret = pp_post_write(ctx)) {
                printf("ret %d, EINVAL %d, ENOMEM %d, EFAULT %d\n", ret, EINVAL, ENOMEM, EFAULT);
                printf("scnt %d, rcnt %d\n",*scnt,*rcnt);
                fprintf(stderr, "Couldn't post write\n");
                return 1;
            }
        }
        break;

    default:
        fprintf(stderr, "Completion for unknown wr_id %d\n",
            (int)wr_id);
        return 1;
    }

    return 0;
}

static void usage(const char *argv0)
{
    printf("Usage:\n");
    printf("  %s            start a server and wait for connection\n", argv0);
    printf("  %s <host>     connect to server at <host>\n", argv0);
    printf("\n");
    printf("Options:\n");
    printf("  -p, --port=<port>      listen on/connect to port <port> (default 18515)\n");
    printf("  -d, --ib-dev=<dev>     use IB device <dev> (default first device found)\n");
    printf("  -i, --ib-port=<port>   use port <port> of IB device (default 1)\n");
    printf("  -s, --size=<size>      size of message to exchange (default 4096)\n");
    printf("  -m, --mtu=<size>       path MTU (default 1024)\n");
    printf("  -r, --rx-depth=<dep>   number of receives to post at a time (default 500)\n");
    printf("  -n, --iters=<iters>    number of exchanges (default 1000)\n");
    printf("  -l, --sl=<sl>          service level value\n");
    printf("  -e, --events           sleep on CQ events (default poll)\n");
    printf("  -g, --gid-idx=<gid index> local port gid index\n");
    printf("  -o, --odp		    use on demand paging\n");
    printf("  -O, --iodp		    use implicit on demand paging\n");
    printf("  -P, --prefetch	    prefetch an ODP MR\n");
    printf("  -t, --ts	            get CQE with timestamp\n");
    printf("  -c, --chk	            validate received buffer\n");
    printf("  -j, --dm	            use device memory\n");
    printf("  -N, --new_send            use new post send WR API\n");
}

int main(int argc, char *argv[])
{
    struct ibv_device      **dev_list;
    struct ibv_device	*ib_dev;
    struct pingpong_context *ctx;
    struct pingpong_dest     my_dest;
    struct pingpong_dest    *rem_dest;
    struct timeval           start, end;
    char                    *ib_devname = NULL;
    char                    *servername = NULL;
    unsigned int             port = 18515;
    int                      ib_port = 1;
    unsigned int             size = 4096;
    enum ibv_mtu		 mtu = IBV_MTU_1024;
    unsigned int             rx_depth = 500;
    unsigned int             iters = 1000;
    int                      use_event = 0;
    int                      routs;
    int                      rcnt, scnt;
    int                      num_cq_events = 0;
    int                      sl = 0;
    int			 gidx = -1;
    char			 gid[33];
    struct ts_params	 ts;

    srand48(getpid() * time(NULL));

    while (1) {
        int c;
        // Any Options not supported
        static struct option long_options[] = {
            { .name = "port",     .has_arg = 1, .val = 'p' },
            { .name = "ib-dev",   .has_arg = 1, .val = 'd' },
            { .name = "ib-port",  .has_arg = 1, .val = 'i' },
            { .name = "size",     .has_arg = 1, .val = 's' },
            { .name = "mtu",      .has_arg = 1, .val = 'm' },
            { .name = "rx-depth", .has_arg = 1, .val = 'r' },
            { .name = "iters",    .has_arg = 1, .val = 'n' },
            { .name = "sl",       .has_arg = 1, .val = 'l' },
            { .name = "events",   .has_arg = 0, .val = 'e' },
            { .name = "gid-idx",  .has_arg = 1, .val = 'g' },
            { .name = "odp",      .has_arg = 0, .val = 'o' },
            { .name = "iodp",     .has_arg = 0, .val = 'O' },
            { .name = "prefetch", .has_arg = 0, .val = 'P' },
            { .name = "ts",       .has_arg = 0, .val = 't' },
            { .name = "chk",      .has_arg = 0, .val = 'c' },
            { .name = "dm",       .has_arg = 0, .val = 'j' },
            { .name = "new_send", .has_arg = 0, .val = 'N' },
            {}
        };

        c = getopt_long(argc, argv, "p:d:i:s:m:r:n:l:eg:oOPtcjN",
                long_options, NULL);

        if (c == -1)
            break;

        switch (c) {
        case 'p':
            port = strtoul(optarg, NULL, 0);
            if (port > 65535) {
                usage(argv[0]);
                return 1;
            }
            break;

        case 'd':
            ib_devname = strdupa(optarg);
            break;

        case 'i':
            ib_port = strtol(optarg, NULL, 0);
            if (ib_port < 1) {
                usage(argv[0]);
                return 1;
            }
            break;

        case 's':
            size = strtoul(optarg, NULL, 0);
            break;

        case 'm':
            mtu = pp_mtu_to_enum(strtol(optarg, NULL, 0));
            if (mtu == 0) {
                usage(argv[0]);
                return 1;
            }
            break;

        case 'r':
            rx_depth = strtoul(optarg, NULL, 0);
            break;

        case 'n':
            iters = strtoul(optarg, NULL, 0);
            break;

        case 'l':
            sl = strtol(optarg, NULL, 0);
            break;

        case 'e':
            ++use_event;
            break;

        case 'g':
            gidx = strtol(optarg, NULL, 0);
            break;

        case 'o':
            use_odp = 1;
            break;
        case 'P':
            prefetch_mr = 1;
            break;
        case 'O':
            use_odp = 1;
            implicit_odp = 1;
            break;
        case 't':
            use_ts = 1;
            break;
        case 'c':
            validate_buf = 1;
            break;

        case 'j':
            use_dm = 1;
            break;

        case 'N':
            use_new_send = 1;
            break;

        default:
            usage(argv[0]);
            return 1;
        }
    }

    if (optind == argc - 1)
        servername = strdupa(argv[optind]);
    else if (optind < argc) {
        usage(argv[0]);
        return 1;
    }

    if (use_odp && use_dm) {
        fprintf(stderr, "DM memory region can't be on demand\n");
        return 1;
    }

    if (!use_odp && prefetch_mr) {
        fprintf(stderr, "prefetch is valid only with on-demand memory region\n");
        return 1;
    }

    if (use_ts) {
        ts.comp_recv_max_time_delta = 0;
        ts.comp_recv_min_time_delta = 0xffffffff;
        ts.comp_recv_total_time_delta = 0;
        ts.comp_recv_prev_time = 0;
        ts.last_comp_with_ts = 0;
        ts.comp_with_time_iters = 0;
    }

    page_size = sysconf(_SC_PAGESIZE);

    dev_list = ibv_get_device_list(NULL);
    if (!dev_list) {
        perror("Failed to get IB devices list");
        return 1;
    }

    if (!ib_devname) {
        ib_dev = *dev_list;
        if (!ib_dev) {
            fprintf(stderr, "No IB devices found\n");
            return 1;
        }
    } else {
        int i;
        for (i = 0; dev_list[i]; ++i)
            if (!strcmp(ibv_get_device_name(dev_list[i]), ib_devname))
                break;
        ib_dev = dev_list[i];
        if (!ib_dev) {
            fprintf(stderr, "IB device %s not found\n", ib_devname);
            return 1;
        }
    }

    ctx = pp_init_ctx(ib_dev, size, rx_depth, ib_port, use_event);
    if (!ctx)
        return 1;

    routs = pp_post_recv(ctx, ctx->rx_depth);
    if (routs < ctx->rx_depth) {
        fprintf(stderr, "Couldn't post receive (%d)\n", routs);
        return 1;
    }

    if (use_event)
        if (ibv_req_notify_cq(pp_cq(ctx), 0)) {
            fprintf(stderr, "Couldn't request CQ notification\n");
            return 1;
        }


    if (pp_get_port_info(ctx->context, ib_port, &ctx->portinfo)) {
        fprintf(stderr, "Couldn't get port info\n");
        return 1;
    }

    my_dest.lid = ctx->portinfo.lid;
    if (ctx->portinfo.link_layer != IBV_LINK_LAYER_ETHERNET &&
                            !my_dest.lid) {
        fprintf(stderr, "Couldn't get local LID\n");
        return 1;
    }

    if (gidx >= 0) {
        if (ibv_query_gid(ctx->context, ib_port, gidx, &my_dest.gid)) {
            fprintf(stderr, "can't read sgid of index %d\n", gidx);
            return 1;
        }
    } else
        memset(&my_dest.gid, 0, sizeof my_dest.gid);

    my_dest.qpn1 = ctx->qp1->qp_num;
    my_dest.qpn2 = ctx->qp2->qp_num;
    my_dest.psn1 = lrand48() & 0xffffff;
    my_dest.psn2 = lrand48() & 0xffffff;
    my_dest.addr1 = ctx->mr1->addr;
    my_dest.addr2 = ctx->mr2->addr;
    my_dest.rkey1 = ctx->mr1->rkey;
    my_dest.rkey2 = ctx->mr2->rkey;
    inet_ntop(AF_INET6, &my_dest.gid, gid, sizeof gid);
    printf("  local address:  LID 0x%04x, QPN1 0x%06x, QPN2 0x%06x, PSN1 0x%06x, PSN2 0x%06x, GID %s\n",
           my_dest.lid, my_dest.qpn1, my_dest.qpn2,
           my_dest.psn1, my_dest.psn2, gid);
    printf("  local addr1 %014p, rkey1 %x, addr2 %014p, rkey2 %x\n", 
           my_dest.addr1, my_dest.rkey1, my_dest.addr2, my_dest.rkey2);


    if (servername)
        rem_dest = pp_client_exch_dest(servername, port, &my_dest);
    else
        rem_dest = pp_server_exch_dest(ctx, ib_port, mtu, port, sl,
                                &my_dest, gidx);

    if (!rem_dest)
        return 1;

    printf("socket complete\n");
    inet_ntop(AF_INET6, &rem_dest->gid, gid, sizeof gid);
    printf("  remote address: LID 0x%04x, QPN1 0x%06x, QPN2 0x%06x, PSN1 0x%06x, PSN2 0x%06x, GID %s\n",
           rem_dest->lid, rem_dest->qpn1, rem_dest->qpn2,
           rem_dest->psn1, rem_dest->psn2, gid);
    printf("  remote addr1 %014p, rkey1 %x, addr2 %014p, rkey2 %x\n", 
           rem_dest->addr1, rem_dest->rkey1, rem_dest->addr2, rem_dest->rkey2);

    ctx->addr1 = rem_dest->addr1;
    ctx->addr2 = rem_dest->addr2;
    ctx->rkey1 = rem_dest->rkey1;
    ctx->rkey2 = rem_dest->rkey2;

    if (servername)
        if (pp_connect_ctx(ctx, ib_port, my_dest.psn1, my_dest.psn2, mtu, sl, rem_dest,
                    gidx))
            return 1;

    ctx->pending = PP_WRITE_WRID;

    rcnt = scnt = 0;

    if (servername) {
        if (validate_buf)
            for (int i = 0; i < size; i += page_size)
                ctx->buf1[i] = i / page_size % sizeof(char);

        int ret;
        if (ret = pp_post_write(ctx)) {
            printf("%d | %d | %d | %d\n", ret, EINVAL, ENOMEM, EFAULT);
            fprintf(stderr, "Couldn't post write\n");
            return 1;
        }
        if (pp_post_read(ctx)) {
            fprintf(stderr, "Couldn't post read\n");
            return 1;
        }
    }

    if (gettimeofday(&start, NULL)) {
        perror("gettimeofday");
        return 1;
    }

    while (rcnt < iters || scnt < iters) {
        int ret;
        int ne, i;
        struct ibv_wc wc[2];

        do {
            ne = ibv_poll_cq(pp_cq(ctx), 2, wc);
            if (ne < 0) {
                fprintf(stderr, "poll CQ failed %d\n", ne);
                return 1;
            }
        } while (!use_event && ne < 1);

        for (i = 0; i < ne; ++i) {
            ret = parse_single_wc(ctx, &scnt, &rcnt, &routs,
                            iters,
                            wc[i].wr_id,
                            wc[i].status,
                            0, &ts);
            if (ret) {
                fprintf(stderr, "parse WC failed %d\n", ne);
                return 1;
            }
        }
    }

    if (gettimeofday(&end, NULL)) {
        perror("gettimeofday");
        return 1;
    }

    {
        float usec = (end.tv_sec - start.tv_sec) * 1000000 +
            (end.tv_usec - start.tv_usec);
        long long bytes = (long long) size * iters * 2;

        printf("%lld bytes in %.2f seconds = %.2f Mbit/sec\n",
               bytes, usec / 1000000., bytes * 8. / usec);
        printf("%d iters in %.2f seconds = %.2f usec/iter\n",
               iters, usec / 1000000., usec / iters);

        if (use_ts && ts.comp_with_time_iters) {
            printf("Max receive completion clock cycles = %" PRIu64 "\n",
                   ts.comp_recv_max_time_delta);
            printf("Min receive completion clock cycles = %" PRIu64 "\n",
                   ts.comp_recv_min_time_delta);
            printf("Average receive completion clock cycles = %f\n",
                   (double)ts.comp_recv_total_time_delta / ts.comp_with_time_iters);
        }

        if ((!servername) && (validate_buf)) {
            for (int i = 0; i < size; i += page_size)
                if (ctx->buf1[i] != i / page_size % sizeof(char))
                    printf("invalid data in page %d\n",
                           i / page_size);
        }
    }

    ibv_ack_cq_events(pp_cq(ctx), num_cq_events);

    if (pp_close_ctx(ctx))
        return 1;

    ibv_free_device_list(dev_list);
    free(rem_dest);

    return 0;
}

```



---

*By CQzhangyu*

