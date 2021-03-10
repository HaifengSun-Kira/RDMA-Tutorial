# Shared example files
cmake_minimum_required(VERSION 3.9)

add_library(rdmacm_tools STATIC
  common.c
  )

add_executable(cmtime cmtime.c)
target_link_libraries(cmtime LINK_PRIVATE rdmacm pthread rdmacm_tools ibverbs)

add_executable(mckey mckey.c)
target_link_libraries(mckey LINK_PRIVATE rdmacm pthread rdmacm_tools ibverbs)

add_executable(rcopy rcopy.c)
target_link_libraries(rcopy LINK_PRIVATE rdmacm rdmacm_tools ibverbs)

add_executable(rdma_client rdma_client.c)
target_link_libraries(rdma_client LINK_PRIVATE rdmacm ibverbs)

add_executable(rdma_server rdma_server.c)
target_link_libraries(rdma_server LINK_PRIVATE rdmacm ibverbs)

add_executable(rdma_xclient rdma_xclient.c)
target_link_libraries(rdma_xclient LINK_PRIVATE rdmacm ibverbs)

add_executable(rdma_xserver rdma_xserver.c)
target_link_libraries(rdma_xserver LINK_PRIVATE rdmacm ibverbs)

add_executable(riostream riostream.c)
target_link_libraries(riostream LINK_PRIVATE rdmacm rdmacm_tools ibverbs)

add_executable(rping rping.c)
target_link_libraries(rping LINK_PRIVATE rdmacm pthread rdmacm_tools ibverbs)

add_executable(rstream rstream.c)
target_link_libraries(rstream LINK_PRIVATE rdmacm rdmacm_tools ibverbs)

add_executable(ucmatose cmatose.c)
target_link_libraries(ucmatose LINK_PRIVATE rdmacm rdmacm_tools ibverbs)

add_executable(udaddy udaddy.c)
target_link_libraries(udaddy LINK_PRIVATE rdmacm rdmacm_tools ibverbs)

add_executable(udpong udpong.c)
target_link_libraries(udpong LINK_PRIVATE rdmacm rdmacm_tools ibverbs)
