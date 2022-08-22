[TOC]

# 容器网络

## 容器单主机网络

> Linux容器能看见的“网络栈”，实际上是被隔离在它自己的Network当中。网络栈包括了：网卡（Network Interface）、回环设备（Loopback Device）、路由器（Routing Table）和iptables规则。

直接使用宿主机网络栈的方式，虽然可以为容器提供良好的网络性能，但也会不可避免地引入共享网络资源的问题，比如端口冲突。所以，大多数情况下，我们都希望容器进程能使用自己Network Namespace里的网络栈，即：**拥有属于自己的IP地址和端口**。

### 容器间通信

如果你想要实现两台主机之间的通信，最直接的办法，就是把它们用一根网线连接起来；而如果你想要实现多台主机之间的通信，那就需要用网线，把它们连接在一台交换机上。

在Linux中，能够起到虚拟交换机作用的网络设备，是网桥（Bridge）。它是一个工作在数据链路层（Data Link）的设备，主要功能是根据MAC地址学习来将数据包转发到网桥的不同端口（Port）上。

### Veth Pair和网桥

> Docker项目会默认在宿主机上创建一个名叫docker0的网桥，凡是连接在docker0网桥上的容器，就可以通过它来进行通信。

Veth Pair设备特点：它被创建后，总是以两张虚拟网卡（Veth Peer）的形式成对出现的。并且，从其中一个“网卡”发出的数据包，可以直接出现在与它对应的另一张“网卡”上，甚至是在两个不同的Network Namespace里。

Veth Pair常常被用作连接不同的Network Namespace的“网线”。

![img](https://cdn.jsdelivr.net/gh/Bruce0hh/Bruce0hh.github.io/pic-bed/e0d28e0371f93af619e91a86eda99a66.png)

1. 虚拟网卡从被接入网桥后，从而成为端口，将所有流入的数据包交给网桥docker0处理。
2. docker0接受Container1的ARP广播，找到MAC地址，并回复给Container1。
3. Container1知道MAC地址后，发送到虚拟网卡veth9c02e56上，然后交给docker0处理。
4. docker0在它的CAM表找到对应端口：vethb4963f3，然后发送给这个端口。

同理，在默认情况下，被限制在Network Namespace里的容器进程，实际上是通过**Veth Pair设备+宿主机网桥**的方式，实现了同其它容器的数据交换。

### 总结

- 容器要想跟外界进行通信，它所发出的IP包就必须从它的Network Namespace里出来，到宿主机上。通过为容器创建一个一端在容器里充当默认网卡、另一端在宿主机上的Veth Pair设备，来解决上述问题。



## 容器跨主机网络

### UDP模式

![img](https://static001.geekbang.org/resource/image/83/6c/8332564c0547bf46d1fbba2a1e0e166c.jpg)

1. flannel0设备类型是TUN设备（Tunnel设备）。功能是**在操作系统内核和用户应用程序之间传递IP包**。
2. 当操作系统将一个IP包发送给flannel0设备之后，flannel0就会把这个IP包，交给创建这个设备的应用程序，也就是Flannel进程。这是一个从内核态（Linux操作系统）向用户态（Flannel进程）的流动方向。
3. 反之，如果Flannel进程向flannel0设备发送了一个IP包，那么这个IP包就会出现在宿主机网络栈中，然后根据宿主机的路由表进行下一步处理。这是一个从用户态向内核态的流动方向。
4. 在由Flannel管理的容器网络里，一台宿主机上的所有容器，都属于该宿主机被分配的一个“子网”。这些**子网与宿主机的对应关系，都保存在Etcd中**。
5. flanneld在收到container-1发给container-2的IP包之后，就会把这个IP包直接封装在一个UDP包里，然后发送给Node 2。
6. 每台宿主机上的flanneld，都监听着一个8285端口，所以flanneld只要把UDP包发往Node 2的8285端口即可。解析UDP包，然后flanneld会直接把这个IP包发送给它所管理的TUN设备，即flannel0设备。
7. 最后根据本机的路由表将IP包转发给docker0网桥。后续部分同单主机网络处理。

![img](https://cdn.jsdelivr.net/gh/Bruce0hh/Bruce0hh.github.io/pic-bed/84caa6dc3f9dcdf8b88b56bd2e22138d.png)

三次用户态和内核态的数据拷贝：

1. 用户态的容器进程发出的IP包经过docker0网桥进入内核态；
2. IP包根据路由表进入TUN（flannel0）设备，从而回到用户态的flanneld进程；
3. flanneld进行UDP封包之后重新进入内核态，将UDP包通过宿主机的eth0发出去。

> **我们在进行系统级编程的时候，有一个非常重要的优化原则，就是要减少用户态到内核态的切换次数，并且把核心的处理逻辑都放在内核态进行**。

### VXLAN（Virtual Extensible LAN，虚拟可扩展局域网）

#### 设计思想

在现有的三层网络之上，“覆盖”一层虚拟的、由内核VXLAN模块负责维护的二层网络，使得连接在这个VXLAN二层网络上的“主机”（虚拟机或者容器都可以）之间，可以像在同一个局域网（LAN）里那样自由通信。当然，实际上，这些“主机”可能分布在不同的宿主机上，甚至是分布在不同的物理机房里。

而为了能够在二层网络上打通“隧道”，VXLAN会在宿主机上设置一个特殊的网络设备作为“隧道”的两端。这个设备就叫作VTEP，即：VXLAN Tunnel End Point（虚拟隧道端点）。

而VTEP设备的作用，其实跟前面的flanneld进程非常相似。只不过，它进行封装和解封装的对象，是二层数据帧（Ethernet frame）；而且这个工作的执行流程，全部是在内核里完成的（因为VXLAN本身就是Linux内核中的一个模块）。

#### 步骤

![img](https://cdn.jsdelivr.net/gh/Bruce0hh/Bruce0hh.github.io/pic-bed/03185fab251a833fef7ed6665d5049f5.jpg)

1. 每台宿主机上名叫flannel.1的设备，就是VXLAN所需的VTEP设备，它既有IP地址，也有MAC地址。
2. “源VTEP设备”收到“原始IP包”后，就要想办法把“原始IP包”加上一个目的MAC地址，封装成一个二层数据帧，然后发送给“目的VTEP设备”（当然，这么做还是因为这个IP包的目的地址不是本机）。
3. 接着需要使用ARP记录，这也是flanneld进程在Node 2节点启动时，自动添加在Node 1上的。Flannel会在每台节点启动时把它的VTEP设备对应的ARP记录，直接下放到其他每台宿主机上。
4. Linux内核还需要再把“内部数据帧”进一步封装成为宿主机网络里的一个普通的数据帧，好让它“载着”“内部数据帧”，通过宿主机的eth0网卡进行传输。
5. Linux内核会在“内部数据帧”前面，加上一个特殊的VXLAN头，用来表示这个“乘客”实际上是一个VXLAN要使用的数据帧。
6. VXLAN头里有一个重要的标志叫作**VNI**，它是VTEP设备识别某个数据帧是不是应该归自己处理的重要标识。而在Flannel中，VNI的默认值是1，这也是为何，宿主机上的VTEP设备都叫作flannel.1的原因，这里的“1”，其实就是VNI的值。
7. **然后，Linux内核会把这个数据帧封装进一个UDP包里发出去**。flannel.1设备实际上要扮演一个“网桥”的角色，在二层网络进行UDP包的转发。而在Linux内核里面，“网桥”设备进行转发的依据，来自于一个叫作FDB（Forwarding Database）的转发数据库。
8. UDP包是一个四层数据包，所以Linux内核会在它前面加上一个IP头，即图中的Outer IP Header，组成一个IP包。并且，在这个IP头里，会填上前面通过FDB查询出来的目的主机的IP地址。
9. Node 2的内核网络栈会发现这个数据帧里有VXLAN Header，并且VNI=1。所以Linux内核会对它进行拆包，拿到里面的内部数据帧，然后根据VNI的值，把它交给Node 2上的flannel.1设备。

![img](https://static001.geekbang.org/resource/image/8c/85/8cede8f74a57617494027ba137383f85.jpg)



> 本文参考《深入剖析Kubernetes》——张磊