---
title: Computer Network Basis
categories: Computer Network
---
# Open System Interconnection Reference Model(开放系统互连参考模型)

自顶向下

## Application Layer(应用层)

包含用户需要的各种协议，比如HTTP。

## Presentation Layer(表示层)

负责统一和管理数据的语法和语义。

## Session Layer(会话层)

允许不同机器上的用户建立会话

## Transport Layer(传输层)

发送时，接收会话层的数据并进行处理，然后传递给网络层，并且确保数据能够正确的传输至另一端，

## Network Layer(网络层)

控制子网的运行，承担拥塞控制的大部分任务。

## Data Line Layer(数据链路层)

负责将原始的物理传输设施转换成一条没有传输错误的软件线路，也就是需要处理传输错误，承担一部分流量控制的任务。

## Physical Layer(物理层)

负责在一条物理通信信道上传输原始二进制数据流。

# TCP/IP Reference Model(TCP/IP参考模型)

自顶向下

## Transport Layer(传输层)

对应开放系统互连模型的传输层

## Internet Layer(互联网层)

对应开放系统互联模型的网络层

## Link Layer(链路层)

大致对应开放系统互联模型的链路层，但是还包含一部分物理层。

# 传输控制协议(Transport Control Protocol, TCP)

为了在不可靠的互联网络上提供可靠的端到端字节流而专门设计的一个传输协议。

## 初次建立通信连接的的过程

1. 客户端向服务器发送请求报文段，SYN标志位为1，并且包含一个随机的起始序号seq。
2. 服务器接收到请求报文段，并同意建立连接以后，就回复客户端一个确认报文段，SYN标志位和ACK标志位都为1，还有个确认号字段为客户端起始序号+1(seq+1)，并且包含一个随机起始序号。
3. 客户端收到确认报文段以后，向服务器发出确认报文，并且给该连接分配缓存和变量。ACK标志位为1，起始序号字段为客户端起始序号+1，服务端确认号ACK为服务端起始序号+1。
4. 建立连接完成，开始传输。

## 断开通信连接的过程

1. 客户端关闭连接时，向服务器发送释放连接报文段，请求释放连接。
2. 服务器收到该报文段后，回复确认报文段。（为了回复客户端我收到请求了，但此时并未开始释放，为的是传输剩余的数据）
3. 服务器数据传输完毕时，向客户端发出请求释放连接报文段。
4. 客户端收到释放报文段后，回复一个确认报文段。服务端接收后会释放连接，客户端在一段时间后才会释放。

# Reference

Computer Networks - Andrew S.Tanenbaum, David J.Wetherall, 严伟、潘爱民译