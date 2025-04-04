![pic](acc_westeast/problem.png)

开发变快了，但产品变慢了（东西向流量增加了）

![pic](acc_westeast/ebpf%20solution.png)

在socket之间进行redirect，实现process to process的访问

Ebfp 在kernel实现

![pic](acc_westeast/packet%20flow.png)

BPF

e.g.  tcpdump  内核把所有收到的报文copy到用户态 -port 43

用户可以更好地控制内核态的操作，对bfp的filter进行优化

![pic](acc_westeast/bpf%20prog.png)

在kernel space，app不需要改变

在容器之间实现快速转发

基于ceiling 的service mesh optimization

![pic](acc_westeast/cutthrough%20mode.png)

基于VPP的用户态实现更快的container commucation

![pic](acc_westeast/cutthrough%20mode2.png)
基于share memory（~50%性能提升）

（ceiling 10%~20%性能提升）
![pic](acc_westeast/session%20based%20abstract%20stack%20interface.png)

![pic](acc_westeast/share%20memory%20stack.png)

（感觉有道理，实现起来会有什么问题？）