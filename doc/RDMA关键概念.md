# RDMA 关键概念

## Queue Pair (QP)

**A QP is roughly equivalent to a socket.** The QP needs to be initialized on both sides of the connection. 

* WR (Work Request): A request which was posted by a user to a work queue.
* WQ (Work Queue): One of Send Queue or Receive Queue.

* RR (Receive Request): A WR which was posted to a RQ which describes where incoming data using a *send* or *RDMA write with immediate* opcode is going to be written. `struct ibv_recv_wr`
* SR (Send Request): A WR which was posted to an SQ which describes how much data is going to be transferred, its dirtection, and the way (*send*, *send with immediate*, *RDMA read*, *RDMA write*, *RDMA write with immediate*, RDMA atomic operations). `struct ibv_send_wr`
* RQ (Receive Queue): A Work Queue which holds RRs posted by the user.
* SQ (Send Queue): A Work Queue which holds SRs posted by the user.



### 通信操作类型

1. Send (Send with Immediate) / Receive
   * Send操作允许用户向一个remote QP的接收队列发送数据。接收方事先必须设置好用于接收数据的receive buffer。
   * Send with Immediate操作，允许用户在发送请求中添加4字节的立即数（ `ibv_send_wr.imm_data` ）。这个立即数会作为receive notification的一部分，不会包含在接收方的data buffer中。
* Receive操作，与Send操作想对应，通知用户data buffer中接收到了数据（可能含有立即数）。
2. RDMA Read / RDMA Write (RDMA Write with Immediate)
   * RDMA Read操作，从远程主机读取一部分内存。 调用者指定远程虚拟地址以及要复制到的本地内存地址。 在执行RDMA操作之前，远程主机必须提供适当的权限来访问其内存。 一旦设置了这些权限，本地就可以进行RDMA read操作，而不会向远程主机发出任何通知。 对于RDMA read和write，远程端都不知道此操作已完成（除了准备权限和资源外）。
   * RDMA Write操作，类似于RDMA read，但是数据被写入远程主机。 执行RDMA write操作时不会通知远程主机。 但是，具有立即数的RDMA write操作会将立即数通知给远程主机。
3. Atomic Fetch and Add / Atomic Compare and Swap



### 传输模式 (QP类型)

![image-20210310153648766](/Users/kira/PKU/RDMA/RDMA-Tutorial/imgs/image-20210310153648766.png)

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

  Ordering and delivery are not guaranteed, and delivered packets may be dropped by the receiver.

  **Multicast** messages are supported (one to many).

  A UD connection is very **similar to a UDP connection**.



## 其他概念

**CQ (Completion Queue)** `struct ibv_cq`

A CQ is an FIFO which contains the completed WRs which were posted to the WQs. 

Every completion says that a specific WR was completed (both successfully completed WRs and unsuccessfully completed WRs).

A CQ is a mechanism to notify the application about information of ended WRs (status, opcode, size, source).

CQs have *n* Completion Queue Entries (CQE). The number of CQEs is specified when the CQ is created. 

When a CQE is polled, it is removed from the CQ.



**MR (Memory Registration)** `struct ibv_mr`

MR is a mechanism that allows an application to describe a set of virtually contiguous memory locations or a set of physically contguous memory locations to the netowrk adapter as a virtually contiguous buffer using Virtual Addresses.

When registering memory, **permissions** are set for the region. Permissions are *local write, remote read, remote write, atomic, and bind*.

**Every MR has a remote and a local key (r_key, l_key)**. Local keys are used by the local HCA to access local memory, such as during a receive operation. Remote keys are given to the remote HCA to allow a remote process access to system memory during RDMA operations.

The same memory buffer can be registered several times (even with different access permissions) and every registration results in a different set of keys.



**PD (Protection Domain)** `struct ibv_pd`

Object whose components can interact with only each other. These components can be **AH, QP, MR, and SRQ**.

**A PD is used to associate QPs with Memory Regions and Memory Windows**, as a means for enabling and controlling network adapter access to Host System memory.

PDs are also used to associate UD queue pairs with Address Handles, as a means of controlling access to UD destinations.



**Asynchronous Events**

The network adapter may send async events to inform the SW about events that occurred in the system.

There are two types of async events:

* Affiliated events: events that occurred to personal objects (CQ, QP, SRQ). Those events will be sent to a specific process.
* Unaffiliated events: events that occurred to global objects (network adapter, port error). Those events will be sent to all processes.



**Scatter Gather Elements** `struct ibv_sge` 

Data is being gathered/scattered using scatter gather elements, which include:

* Address: address of the local data buffer that the data will be gathered from or scattered to. 
* Size: the size of the data that will be read from / written to this address.
* L_key: the local key of the MR that was registered to this buffer.



**MW (Memory Window)**

An MW allows the application to have more flexible control over remote access to its memory. Memory Windows are intended for situations where the application:

- wants to grant and revoke remote access rights to a registered Region in a dynamic fashion with less of a performance penalty than using deregistration/registration or reregistration.

- wants to grant different remote access rights to different remote agents and/or grant those rights over different ranges within a registered Region.

  The operation of associating an MW with an MR is called Binding.
  