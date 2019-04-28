作者：惨绿少年Linux，马哥Linux原创作者社群特约作者，资深Linux运维工程师，作者博客：[www.nmtui.com](https://link.jianshu.com/?t=http%3A%2F%2Fwww.nmtui.com)，擅长虚拟化、OpenStack等前沿技术。

[LVS简快入门实战]: https://www.jianshu.com/p/fb5cceb0bef4

## **1.1 负载均衡介绍**

------

### **1.1.1 负载均衡的妙用**

负载均衡（Load Balance）集群提供了一种廉价、有效、透明的方法，来扩展网络设备和服务器的负载、带宽、增加吞吐量、加强网络数据处理能力、提高网络的灵活性和可用性。

单台计算机无法承受大规模的并发访问或数据流量了，此时需要搭建负载均衡集群把流量分摊到多台节点设备上分别处理，即减少用户等待响应的时间又提升了用户体验；

7*24小时的服务保证，任意一个或多个有限后端节点设备宕机，不能影响整个业务的运行。

### **1.1.2 为什么要用lvs**

工作在网络模型的7层，可以针对http应用做一些分流的策略，比如针对域名、目录结构，Nginx单凭这点可利用的场合就远多于LVS了。

最新版本的Nginx也支持4层TCP负载，曾经这是LVS比Nginx好的地方。

Nginx对网络稳定性的依赖非常小，理论上能ping通就就能进行负载功能，这个也是它的优势之一，相反LVS对网络稳定性依赖比较大。

Nginx安装和配置比较简单，测试起来比较方便，它基本能把错误用日志打印出来。LVS的配置、测试就要花比较长的时间了，LVS对网络依赖比较大。

那为什么要用lvs呢？

简单一句话，当并发超过了Nginx上限，就可以使用LVS了。

日1000-2000W PV或并发请求1万以下都可以考虑用Nginx。

大型门户网站，电商网站需要用到LVS。

## **1.2 LVS介绍**

------

LVS是Linux Virtual Server的简写，意即Linux虚拟服务器，是一个虚拟的服务器集群系统，可以在UNIX/LINUX平台下实现负载均衡集群功能。该项目在1998年5月由章文嵩博士组织成立，是中国国内最早出现的自由软件项目之一。

### **1.2.1 相关参考资料**

LVS官网：[http://www.linuxvirtualserver.org/index.html](https://link.jianshu.com?t=http%3A%2F%2Fwww.linuxvirtualserver.org%2Findex.html)

相关中文资料

LVS项目介绍           [http://www.linuxvirtualserver.org/zh/lvs1.html](https://link.jianshu.com?t=http%3A%2F%2Fwww.linuxvirtualserver.org%2Fzh%2Flvs1.html)

LVS集群的体系结构     [http://www.linuxvirtualserver.org/zh/lvs2.html](https://link.jianshu.com?t=http%3A%2F%2Fwww.linuxvirtualserver.org%2Fzh%2Flvs2.html)

LVS集群中的IP负载均衡技术  [http://www.linuxvirtualserver.org/zh/lvs3.html](https://link.jianshu.com?t=http%3A%2F%2Fwww.linuxvirtualserver.org%2Fzh%2Flvs3.html)

LVS集群的负载调度      [http://www.linuxvirtualserver.org/zh/lvs4.html](https://link.jianshu.com?t=http%3A%2F%2Fwww.linuxvirtualserver.org%2Fzh%2Flvs4.html)

### 1.2.2 LVS内核模块ip_vs介绍

早在2.2内核时， IPVS就已经以内核补丁的形式出现。

从2.4.23版本开始，IPVS软件就合并到Linux内核的常用版本的内核补丁的集合。

从2.4.24以后IPVS已经成为Linux官方标准内核的一部分。

![img](assets\9721977-40dc2a0acf997158.webp)

LVS无需安装

安装的是管理工具，第一种叫ipvsadm，第二种叫keepalive

ipvsadm是通过命令行管理，而keepalive读取配置文件管理

后面我们会用Shell脚本实现keepalive的功能

## **1.3 LVS集群搭建**

------

### 1.3.1 集群环境说明



![img](assets\9721977-971952a284c72609.webp)



主机说明

```
 [root@lb03 ~]# cat /etc/redhat-release
 CentOS Linux release 7.4.1708 (Core)
 [root@lb03 ~]# uname -a
 Linux lb03 3.10.0-693.el7.x86_64 #1 SMP Tue Aug 22 21:09:27 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux
 [root@lb03 ~]# systemctl status firewalld.service
 ● firewalld.service - firewalld - dynamic firewall daemon
 Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
 Active: inactive (dead)
 Docs: man:firewalld(1)
 [root@lb03 ~]# getenforce
 Disabled
```

web环境说明

```
 [root@lb03 ~]# curl 10.0.0.17
 web03
 [root@lb03 ~]# curl 10.0.0.18
 web04
```

web服务器的搭建参照：

Tomcat： [http://www.cnblogs.com/clsn/p/7904611.html](https://link.jianshu.com?t=http%3A%2F%2Fwww.cnblogs.com%2Fclsn%2Fp%2F7904611.html)

Nginx： [http://www.cnblogs.com/clsn/p/7750615.html](https://link.jianshu.com?t=http%3A%2F%2Fwww.cnblogs.com%2Fclsn%2Fp%2F7750615.html)



### 1.3.2 安装ipvsadm管理工具

安装管理工具

```
yum -y install ipvsadm
```



查看当前LVS状态，顺便激活LVS内核模块。

```
ipvsadm
```



查看系统的LVS模块。

```
 [root@lb03 ~]# lsmod|grep ip_vs
 ip_vs_wrr              12697  1
 ip_vs                 141092  3 ip_vs_wrr
 nf_conntrack          133387  1 ip_vs
 libcrc32c              12644  3 xfs,ip_vs,nf_conntrack
```



### 1.3.3 LVS集群搭建

配置LVS负载均衡服务（在lb03操作）

- [ ] 步骤1：在eth0网卡绑定VIP地址（ip）
- [ ] 步骤2：清除当前所有LVS规则（-C）
- [ ] 步骤3：设置tcp、tcpfin、udp链接超时时间（--set）
- [ ] 步骤4：添加虚拟服务（-A），-t指定虚拟服务的IP端口，-s 指定调度算法 调度算法见man ipvsadm， rr wrr 权重轮询 -p 指定超时时间
- [ ] 步骤5：将虚拟服务关联到真实服务上（-a） -r指定真实服务的IP端口 -g LVS的模式 DR模式 -w 指定权重
- [ ] 步骤6：查看配置结果（-ln）



命令集：

```
 ip addr add 10.0.0.13/24 dev eth0
 ipvsadm -C
 ipvsadm --set 30 5 60
 ipvsadm -A -t 10.0.0.13:80 -s wrr -p 20
 ipvsadm -a -t 10.0.0.13:80 -r 10.0.0.17:80 -g -w 1
 ipvsadm -a -t 10.0.0.13:80 -r 10.0.0.18:80 -g -w 1
 ipvsadm -ln
```



检查结果：

```
 [root@lb03 ~]# ipvsadm -ln
 IP Virtual Server version 1.2.1 (size=4096)
 Prot LocalAddress:Port Scheduler Flags
 -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
 TCP  10.0.0.13:80 wrr persistent 20
 -> 10.0.0.17:80                 Route   1      0          0
 -> 10.0.0.18:80                 Route   1      0          0
```



ipvsadm参数说明：(更多参照 man ipvsadm)



![img](assets\9721977-c92ad7363de5b188.webp)



### 1.3.4 在应用服务器配置操作

- [ ] 步骤1：在lo网卡绑定VIP地址（ip）
- [ ] 步骤2：修改内核参数抑制ARP响应



命令集：

```
ip addr add 10.0.0.13/32 dev lo

cat >>/etc/sysctl.conf<<EOF
 net.ipv4.conf.all.arp_ignore = 1
 net.ipv4.conf.all.arp_announce = 2
 net.ipv4.conf.lo.arp_ignore = 1
 net.ipv4.conf.lo.arp_announce = 2
 EOF
 sysctl -p
```

**至此LVS集群配置完毕！**



### 1.3.5 进行访问测试

浏览器访问：

![img](assets\9721977-f67c9e1b8634c505.webp)



命令行测试：

```
[root@lb04 ~]# curl 10.0.0.13
 web03
```



抓包查看结果：



![img](assets\9721977-f475aaace047b31f.webp)



arp解析查看：

```
[root@lb04 ~]# arp -n
 Address                  HWtype  HWaddress           Flags Mask            Iface
 10.0.0.254               ether   00:50:56:e9:9f:2c   C                     eth0
 10.0.0.18                ether   00:0c:29:ea:ca:55   C                     eth0
 10.0.0.13                ether   00:0c:29:de:7c:97   C                     eth0
 172.16.1.15              ether   00:0c:29:de:7c:a1   C                     eth1
 10.0.0.17                ether   00:0c:29:4a:ac:4a   C                     eth0
```



## **1.4 负载均衡（LVS）相关名词**



![img](assets\9721977-718e27680f9c5ad9.webp)



术语说明：

```
 DS：Director Server。指的是前端负载均衡器节点。
 RS：Real Server。后端真实的工作服务器。
 VIP：向外部直接面向用户请求，作为用户请求的目标的IP地址。
 DIP：Director Server IP，主要用于和内部主机通讯的IP地址。
 RIP：Real Server IP，后端服务器的IP地址。
 CIP：Client IP，访问客户端的IP地址。
```



### 1.4.1 LVS集群的工作模式--DR直接路由模式

DR模式是通过改写请求报文的目标MAC地址，将请求发给真实服务器的，而真实服务器将响应后的处理结果直接返回给客户端用户。

DR技术可极大地提高集群系统的伸缩性。但要求调度器LB与真实服务器RS都有一块物理网卡连在同一物理网段上，即必须在同一局域网环境。


![img](assets\9721977-a50b8a5e8ba28450.webp)



DR直接路由模式说明：

- a)通过在调度器LB上修改数据包的目的MAC地址实现转发。注意，源IP地址仍然是CIP，目的IP地址仍然是VIP。
-  b)请求的报文经过调度器，而RS响应处理后的报文无需经过调度器LB，因此，并发访问量大时使用效率很高，比Nginx代理模式强于此处。
-  c)因DR模式是通过MAC地址的改写机制实现转发的，因此，所有RS节点和调度器LB只能在同一个局域网中。需要注意RS节点的VIP的绑定(lo:vip/32)和ARP抑制问题。
-  d)强调下：RS节点的默认网关不需要是调度器LB的DIP，而应该直接是IDC机房分配的上级路由器的IP(这是RS带有外网IP地址的情况)，理论上讲，只要RS可以出网即可，不需要必须配置外网IP，但走自己的网关，那网关就成为瓶颈了。
-  e)由于DR模式的调度器仅进行了目的MAC地址的改写，因此，调度器LB无法改变请求报文的目的端口。LVS DR模式的办公室在二层数据链路层（MAC），NAT模式则工作在三层网络层（IP）和四层传输层（端口）。
-  f)当前，调度器LB支持几乎所有UNIX、Linux系统，但不支持windows系统。真实服务器RS节点可以是windows系统。
-  g)总之，DR模式效率很高，但是配置也较麻烦。因此，访问量不是特别大的公司可以用haproxy/Nginx取代之。这符合运维的原则：简单、易用、高效。日1000-2000W PV或并发请求1万以下都可以考虑用haproxy/Nginx(LVS的NAT模式)
-  h)直接对外的访问业务，例如web服务做RS节点，RS最好用公网IP地址。如果不直接对外的业务，例如：MySQL，存储系统RS节点，最好只用内部IP地址。



DR的实现原理和数据包的改变

![img](assets\9721977-c213bc31a08b243f.webp)

- (a) 当用户请求到达Director Server，此时请求的数据报文会先到内核空间的PREROUTING链。 此时报文的源IP为CIP，目标IP为VIP
- (b) PREROUTING检查发现数据包的目标IP是本机，将数据包送至INPUT链
- (c) IPVS比对数据包请求的服务是否为集群服务，若是，将请求报文中的源MAC地址修改为DIP的MAC地址，将目标MAC地址修改RIP的MAC地址，然后将数据包发至POSTROUTING链。 此时的源IP和目的IP均未修改，仅修改了源MAC地址为DIP的MAC地址，目标MAC地址为RIP的MAC地址
-  (d) 由于DS和RS在同一个网络中，所以是通过二层来传输。POSTROUTING链检查目标MAC地址为RIP的MAC地址，那么此时数据包将会发至Real Server。
- (e) RS发现请求报文的MAC地址是自己的MAC地址，就接收此报文。处理完成之后，将响应报文通过lo接口传送给eth0网卡然后向外发出。 此时的源IP地址为VIP，目标IP为CIP
-  (f) 响应报文最终送达至客户端



注：[ipvs负载均衡模块的内核实现](https://blog.csdn.net/dog250/article/details/5846448)

[直接路由方式]：

直接查找路由表，以原始数据包的目的地址为查找键。本地配置的ip地址就是数据包的目的地址，数据既然已经到了本地为何还要查找，为何还要继续路由？这是因为本地的目的地到达情景仅仅是一个假象，真正提供服务的机器还在后面，也就是说服务被负载均衡了。此时问题是，既然本地配置了一个目的地ip地址，其它机器还能配置这个ip地址吗？那样的话岂不ip冲突了吗？

在直接路由模式中，负载均衡器和“后面”真正提供服务的机器都配置有同一个ip地址，在负载均衡器中，该ip配置在一个物理的真实网卡上，用来接收客户端的数据包，很显然，这些数据包最终肯定走到了ip_local_deliver这个函数中，接下来数据包要通过NF_IP_LOCAL_IN这个hook，恰好ipvs等在这里，接着调用ip_vs_in这个hook函数，经过判断发现数据包需要进行负载均衡后，会调用已经建立的到真实机器的连接ip_vs_conn这个数据结构的packet_xmit回调函数，在该函数中会以ip_vs_conn中的信息来查找路由，ip_vs_conn中有三个字段很重要：caddr-客户端的ip；vaddr-虚拟ip，也就是在负载均衡器物理网卡上配置的ip，同时该ip将绑定在所有提供真实服务的所谓均衡机器的loopback网卡上；daddr-这是均衡机器的物理网卡的ip，简单点理解为负载均衡器直连。

数据从查找到的daddr路由的出口设备发送了出去，最终数据到达了这个daddr，然后进入路由查找，看是local-in还是forward，在查找的时候会调用fib_lookup函数，它会查找各个路由表，或者它会遍历各个路由规则表--在配置了MULTIPLE_TABLES的情况下，最终它发现目的ip地址就是本机--绑定在loopback上的地址，也就是vaddr，然后数据被交由上层真正地被处理。之所以可以做到从一个网口进入的本地接收数据包的目的地址并没有配置在该网口上是因为路由查找的默认策略是不检查入口网卡的ip和目的地ip的关系的，这么检查也没有多大意义，因为大多数的过路包的目的ip和本机网卡ip本来就没有什么关系，而在检查路由的伊始还区分不出这是过路包还是本地接收包。不过想实现入口网口和路由的关系绑定也不是不能实现，办法就是配置一条策略路由或者在MULTIPLE_TABLES的情况下添加一个新的规则表，将fib_rule的r_ifindex字段进行硬绑定即可，这样的话fib_lookup中会对其进行检查：(r->r_ifindex && r->r_ifindex != flp->iif)

为了将这些真实的处理机器彻底隐藏，需要隐藏它们的虚拟ip地址，由于所有的机器都配置了同样的vaddr，那么免费arp的发送会导致大量的ip冲突，因此需要做的就是将这些ip隐藏，仅仅开放给进入包的协议栈路由查找。所谓的隐藏就是不让本机协议栈以外的别人知道，由于任何的ip在以太网中都是通过arp来使别人知道自己存在的，你要ping一台局域网的机器，首先要arp一下，得到回复后方知目的地的mac地址，这样才能实际发送，所以只要能够抑制这个虚拟网址发送arp回应即可，它本身配置在loopback上，为了不使它响应外部的任何arp请求，只需要配置一个内核参数即可--arp_ignore，这个参数控制对ip没有配置在收到arp请求的网卡上的请求的回复策略，vaddr配置在loopback上，而arp请求肯定是由ethX进入的，因此这样可以不回应任何arp请求，同时它自己也不会广播免费arp，因此这个vaddr实际上是一个“无用”的ip，无用的意义在于它无法被寻址。

真实的服务器监听哪个ip地址呢？它监听的就是vaddr，就是配置在loopback上的vaddr，这个地址除了负载均衡器可以连到，对于其它主机是不可见的，因为它无法响应arp请求(arp_ignoe)，然而你还是可以用vaddr去连接真实服务器上的服务，前提是设置一条静态路由指向特定的真实服务器，实际上如果设置了静态路由，ping之也是可以通的，之所以设置arp_ignore是怕真实服务器回应arp请求和频发arp广播，很多情况下是没有必要设置它的。

[NAT方式]：

这种方式配置起来比较简单，不需要配置虚拟ip以及arp内核参数之类的，但是性能较直接路由模式就有点不佳了，毕竟要做的事情多了。NAT模式很简单，就是将ip地址和端口信息修改成真实均衡机器们的ip和端口，这样真实的服务机器就被隐藏在负载均衡器后面了，思想和普通的nat是一样的。

[隧道模式]：

隧道模式就是将数据重新打包成一个新的ip数据包，然后可以通过修改代码实现发送到虚拟网卡，由应用程序来做负载均衡，也可以直接发送到一个ipip隧道中去。作者：dog250 





## **1.5 在web端的操作有什么含义？**

------

### 1.5.1 RealServer为什么要在lo接口上配置VIP？

既然要让RS能够处理目标地址为vip的IP包，首先必须要让RS能接收到这个包。

在lo上配置vip能够完成接收包并将结果返回client。

### 1.5.2 在eth0网卡上配置VIP可以吗？

不可以，将VIP设置在eth0网卡上,会影响RS的arp请求,造成整体LVS集群arp缓存表紊乱，以至于整个负载均衡集群都不能正常工作。

### 1.5.3 为什么要抑制ARP响应？

**① arp协议说明**

ARP协议,全称"Address Resolution Protocol",中文名是地址解析协议，使用ARP协议可实现通过IP地址获得对应主机的物理地址(MAC地址)。

ARP协议要求通信的主机双方必须在同一个物理网段（即局域网环境）！

为了提高IP转换MAC的效率，系统会将解析结果保存下来，这个结果叫做ARP缓存。

```
 Windows查看ARP缓存命令 arp -a
 Linux查看ARP缓存命令 arp -n
 Linux解析IP对应的MAC地址 arping -c 1 -I eth0 10.0.0.6
```

ARP缓存表是把双刃剑

a) 主机有了arp缓存表，可以加快ARP的解析速度，减少局域网内广播风暴。因为arp是发广播解析的，频繁的解析也是消耗带宽的，尤其是机器多的时候。

b) 正是有了arp缓存表，给恶意黑客带来了攻击服务器主机的风险，这个就是arp欺骗攻击。

c) 切换路由器，负载均衡器等设备时，可能会导致短时网络中断。因为所有的客户端ARP缓存表没有更新

**②服务器切换ARP问题**

当集群中一台提供服务的lb01机器宕机后，然后VIP会转移到备机lb02上，但是客户端的ARP缓存表的地址解析还是宕机的lb01的MAC地址。从而导致，即使在lb02上添加VIP，也会发生客户端无法访问的情况。

解决办法是：当lb01宕机，VIP地址迁移到lb02时，需要通过arping命令通知所有网络内机器更新本地的ARP缓存表，从而使得客户机访问时重新广播获取MAC地址。

这个是自己开发服务器高可用脚本及所有高可用软件必须考虑到的问题。

ARP广播进行新的地址解析

```
 arping -I eth0 -c 1 -U VIP
 arping -I eth0 -c 1 -U 10.0.0.13
```



测试命令

```
 ip addr del 10.0.0.13/24 dev eth0
 ip addr add 10.0.0.13/24 dev eth0
 ip addr show eth0
 arping -I eth0 -c 1 -U 10.0.0.13
```



windows查看arp -a

```
 接口: 10.0.0.1 --- 0x12
 Internet 地址         物理地址              类型
 10.0.0.13             00-0c-29-de-7c-97     动态
 10.0.0.15             00-0c-29-de-7c-97     动态
 10.0.0.16             00-0c-29-2e-47-20     动态
 10.0.0.17             00-0c-29-4a-ac-4a     动态
 10.0.0.18             00-0c-29-ea-ca-55     动态
```



**③arp_announce\**\**和arp_ignore\**\**详解**

配置的内核参数

```
 net.ipv4.conf.all.arp_ignore = 1
 net.ipv4.conf.all.arp_announce = 2
 net.ipv4.conf.lo.arp_ignore = 1
 net.ipv4.conf.lo.arp_announce = 2
```

lvs在DR模式下需要关闭arp功能

**arp_announce**

对网络接口上，本地IP地址的发出的，ARP回应，作出相应级别的限制:

确定不同程度的限制,宣布对来自本地源IP地址发出Arp请求的接口



![img](assets\9721977-9c5f29e139c38b6f.webp)



**arp_ignore定义**

对目标地定义对目标地址为本地IP的ARP询问不同的应答模式0

![img](assets\9721977-951d329c4769e302.webp)



**抑制RS端arp前的广播情况**

![img](assets\9721977-3076266b85c74abc.webp)



**抑制RS端arp后广播情况**

![img](assets\9721977-ebb3ce49e7ec8532.webp)



## **1.6 LVS集群的工作模式**


`DR（Direct Routing）直接路由模式`

`NAT（Network Address Translation）`

`TUN（Tunneling）隧道模式`

`FULLNAT（Full Network Address Translation）`



### 1.6.1 LVS集群的工作模式--NAT



![img](assets\9721977-148564a7b8179c09.webp)  



通过网络地址转换，调度器LB重写请求报文的目标地址，根据预设的调度算法，将请求分派给后端的真实服务器，真实服务器的响应报文处理之后，返回时必须要通过调度器，经过调度器时报文的源地址被重写，再返回给客户，完成整个负载调度过程。

收费站模式---来去都要经过LB负载均衡器。

NAT方式的实现原理和数据包的改变



![img](assets\9721977-31fa077f833fb3b6.webp)



l RS应该使用私有地址，RS的网关必须指向DIP

- (a). 当用户请求到达Director Server，此时请求的数据报文会先到内核空间的PREROUTING链。 此时报文的源IP为CIP，目标IP为VIP
-  (b). PREROUTING检查发现数据包的目标IP是本机，将数据包送至INPUT链
-  (c). IPVS比对数据包请求的服务是否为集群服务，若是，修改数据包的目标IP地址为后端服务器IP，然后将数据包发至POSTROUTING链。 此时报文的源IP为CIP，目标IP为RIP
-  (d). POSTROUTING链通过选路，将数据包发送给Real Server
-  (e). Real Server比对发现目标为自己的IP，开始构建响应报文发回给Director Server。 此时报文的源IP为RIP，目标IP为CIP
-  (f). Director Server在响应客户端前，此时会将源IP地址修改为自己的VIP地址，然后响应给客户端。 此时报文的源IP为VIP，目标IP为CIP



**LVS-NAT\**\**模型的特性**

l DIP和RIP必须在同一个网段内

l 请求和响应报文都需要经过Director Server，高负载场景中，Director Server易成为性能瓶颈

l 支持端口映射

l RS可以使用任意操作系统

l 缺陷：对Director Server压力会比较大，请求和响应都需经过director server



### 1.6.2 LVS集群的工作模式--隧道模式TUN



![img](assets\9721977-e26d32c9bdbeaeb7.webp)



采用NAT技术时，由于请求和响应的报文都必须经过调度器地址重写，当客户请求越来越多时，调度器的处理能力将成为瓶颈，为了解决这个问题，调度器把请求的报文通过IP隧道(相当于ipip或ipsec )转发至真实服务器，而真实服务器将响应处理后直接返回给客户端用户，这样调度器就只处理请求的入站报文。由于一般网络服务应答数据比请求报文大很多，采用 VS/TUN技术后，集群系统的最大吞吐量可以提高10倍。

VS/TUN工作流程,它的连接调度和管理与VS/NAT中的一样，只是它的报文转发方法不同。调度器根据各个服务器的负载情况，连接数多少，动态地选择一台服务器，将原请求的报文封装在另一个IP报文中，再将封装后的IP报文转发给选出的真实服务器；真实服务器收到报文后，先将收到的报文解封获得原来目标地址为VIP地址的报文, 服务器发现VIP地址被配置在本地的IP隧道设备上(此处要人为配置)，所以就处理这个请求，然后根据路由表将响应报文直接返回给客户。

**TUN原理和数据包的改变**



![img](assets\9721977-cd88f5eec3322523.webp)



- (a) 当用户请求到达Director Server，此时请求的数据报文会先到内核空间的PREROUTING链。 此时报文的源IP为CIP，目标IP为VIP 。
- (b) PREROUTING检查发现数据包的目标IP是本机，将数据包送至INPUT链
-  (c) IPVS比对数据包请求的服务是否为集群服务，若是，在请求报文的首部再次封装一层IP报文，封装源IP为为DIP，目标IP为RIP。然后发至POSTROUTING链。 此时源IP为DIP，目标IP为RIP
-  (d) POSTROUTING链根据最新封装的IP报文，将数据包发至RS（因为在外层封装多了一层IP首部，所以可以理解为此时通过隧道传输）。 此时源IP为DIP，目标IP为RIP
-  (e) RS接收到报文后发现是自己的IP地址，就将报文接收下来，拆除掉最外层的IP后，会发现里面还有一层IP首部，而且目标是自己的lo接口VIP，那么此时RS开始处理此请求，处理完成之后，通过lo接口送给eth0网卡，然后向外传递。 此时的源IP地址为VIP，目标IP为CIP
   (f) 响应报文最终送达至客户端



l RIP、VIP、DIP全是公网地址LVS-Tun模型特性

l RS的网关不会也不可能指向DIP

l 所有的请求报文经由Director Server，但响应报文必须不能进过Director Server

l 不支持端口映射

l RS的系统必须支持隧道



### 1.6.3 LVS集群的工作模式--FULLNAT



![img](assets\9721977-2d5034b7f0e49e7e.webp)



LVS的DR和NAT模式要求RS和LVS在同一个vlan中，导致部署成本过高；TUNNEL模式虽然可以跨vlan，但RealServer上需要部署ipip隧道模块等，网络拓扑上需要连通外网，较复杂，不易运维。

为了解决上述问题，开发出FULLNAT，该模式和NAT模式的区别是：数据包进入时，除了做DNAT，还做SNAT（用户ip->内网ip），从而实现LVS-RealServer间可以跨vlan通讯，RealServer只需要连接到内网。

类比地铁站多个闸机。



## **1.7 IPVS调度器实现了如下八种负载调度算法：**

------

　　**a) \**\**轮询（Round Robin\**\**）RR**

调度器通过"轮叫"调度算法将外部请求按顺序轮流分配到集群中的真实服务器上，它均等地对待每一台服务器，而不管服务器上实际的连接数和系统负载。

　　**b) \**\**加权轮叫（Weighted Round Robin\**\**）WRR**

调度器通过"加权轮叫"调度算法根据真实服务器的不同处理能力来调度访问请求。这样可以保证处理能力强的服务器处理更多的访问流量。调度器可以自动问询真实服务器的负载情况，并动态地调整其权值。

　　**c) \**\**最少链接（Least Connections\**\**） LC**

调度器通过"最少连接"调度算法动态地将网络请求调度到已建立的链接数最少的服务器上。如果集群系统的真实服务器具有相近的系统性能，采用"最小连接"调度算法可以较好地均衡负载。

　　**d) \**\**加权最少链接（Weighted Least Connections\**\**） Wlc**

在集群系统中的服务器性能差异较大的情况下，调度器采用"加权最少链接"调度算法优化负载均衡性能，具有较高权值的服务器将承受较大比例的活动连接负载。调度器可以自动问询真实服务器的负载情况，并动态地调整其权值。

　　**e) \**\**基于局部性的最少链接（Locality-Based Least Connections\**\**） Lblc**

"基于局部性的最少链接" 调度算法是针对目标IP地址的负载均衡，目前主要用于Cache集群系统。该算法根据请求的目标IP地址找出该目标IP地址最近使用的服务器，若该服务器 是可用的且没有超载，将请求发送到该服务器；若服务器不存在，或者该服务器超载且有服务器处于一半的工作负载，则用"最少链接"的原则选出一个可用的服务 器，将请求发送到该服务器。

　　**f) \**\**带复制的基于局部性最少链接（Locality-Based Least Connections with Replication\**\**）**

"带复制的基于局部性最少链接"调度算法也是针对目标IP地址的负载均衡，目前主要用于Cache集群系统。它与LBLC算法的不同之处是它要维护从一个 目标IP地址到一组服务器的映射，而LBLC算法维护从一个目标IP地址到一台服务器的映射。该算法根据请求的目标IP地址找出该目标IP地址对应的服务 器组，按"最小连接"原则从服务器组中选出一台服务器，若服务器没有超载，将请求发送到该服务器，若服务器超载；则按"最小连接"原则从这个集群中选出一 台服务器，将该服务器加入到服务器组中，将请求发送到该服务器。同时，当该服务器组有一段时间没有被修改，将最忙的服务器从服务器组中删除，以降低复制的 程度。

　　**g) \**\**目标地址散列（Destination Hashing\**\**） Dh**

"目标地址散列"调度算法根据请求的目标IP地址，作为散列键（Hash Key）从静态分配的散列表找出对应的服务器，若该服务器是可用的且未超载，将请求发送到该服务器，否则返回空。

　　**h) \**\**源地址散列（Source Hashing\**\**）SH**

"源地址散列"调度算法根据请求的源IP地址，作为散列键（Hash Key）从静态分配的散列表找出对应的服务器，若该服务器是可用的且未超载，将请求发送到该服务器，否则返回空。



## **1.8 LVS+Keepalived方案实现**

------

### 1.8.1 keepalived功能

1. 添加VIP
2. 添加LVS配置
3. 高可用（VIP漂移）
4. web服务器健康检查
5. 

### 1.8.2 在负载器安装Keepalived软件

```
yum -y install keepalived
```

检查软件是否安装

```
[root@lb03 ~]# rpm -qa keepalived
keepalived-1.3.5-1.el7.x86_64
```



### 1.8.3 修改配置文件

lb03上keepalied配置文件


lb03 /etc/keepalived/keepalived.conf



lb04的Keepalied配置文件

lb04 /etc/keepalived/keepalived.conf



keepalived persistence_timeout参数意义 LVS Persistence 参数的作用

[http://blog.csdn.net/nimasike/article/details/53911363](https://link.jianshu.com?t=http%3A%2F%2Fblog.csdn.net%2Fnimasike%2Farticle%2Fdetails%2F53911363)



### 1.8.4 启动keepalived服务

```
[root@lb03 ~]# systemctl restart  keepalived.service
```


检查lvs状态

```
[root@lb03 ~]# ipvsadm -ln
 IP Virtual Server version 1.2.1 (size=4096)
 Prot LocalAddress:Port Scheduler Flags
 -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
 TCP  10.0.0.13:80 wrr persistent 50
 -> 10.0.0.17:80                 Route   1      0          0
 -> 10.0.0.18:80                 Route   1      0          0
```


检查虚拟ip

```
[root@lb03 ~]# ip a s eth0
 2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
 link/ether 00:0c:29:de:7c:97 brd ff:ff:ff:ff:ff:ff
 inet 10.0.0.15/24 brd 10.0.0.255 scope global eth0
 valid_lft forever preferred_lft forever
 inet 10.0.0.13/24 scope global secondary eth0
 valid_lft forever preferred_lft forever
 inet6 fe80::20c:29ff:fede:7c97/64 scope link
 valid_lft forever preferred_lft forever


```



### 1.8.5 在web服务器上进行配置

（在web03/web04同时操作下面步骤）
 步骤1：在lo网卡绑定VIP地址（ip）
 步骤2：修改内核参数抑制ARP响应



```
 ip addr add 10.0.0.13/32 dev lo

cat >>/etc/sysctl.conf<<EOF
 net.ipv4.conf.all.arp_ignore = 1net.ipv4.conf.all.arp_announce = 2net.ipv4.conf.lo.arp_ignore = 1net.ipv4.conf.lo.arp_announce = 2EOF
 sysctl -p
```



注意：web服务器上的配置为临时生效，可以将其写入rc.local文件，注意文件的执行权限。



使用curl命令进行测试

```
[root@lb04 ~]# curl 10.0.0.13web03
```



**至此keepalived+lvs配置完毕**



## 1.9 常见LVS负载均衡高可用解决方案**

------

开发类似keepalived的脚本，早期的办法，现在不推荐使用。

heartbeat+lvs+ldirectord脚本配置方案，复杂不易控制，不推荐使用

RedHat工具piranha，一个web界面配置LVS。

LVS-DR+keepalived方案，推荐最优方案，简单、易用、高效。



### 1.9.1 lvs排错思路



![img](assets\9721977-6d2825096cf72982.webp)





### 1.9.2 参考文档


LVS项目介绍            [http://www.linuxvirtualserver.org/zh/lvs1.html](https://link.jianshu.com?t=http%3A%2F%2Fwww.linuxvirtualserver.org%2Fzh%2Flvs1.html)
 LVS集群的体系结构      [http://www.linuxvirtualserver.org/zh/lvs2.html](https://link.jianshu.com?t=http%3A%2F%2Fwww.linuxvirtualserver.org%2Fzh%2Flvs2.html)
 LVS集群中的IP负载均衡技术  [http://www.linuxvirtualserver.org/zh/lvs3.html](https://link.jianshu.com?t=http%3A%2F%2Fwww.linuxvirtualserver.org%2Fzh%2Flvs3.html)
 LVS集群的负载调度       [http://www.linuxvirtualserver.org/zh/lvs4.html](https://link.jianshu.com?t=http%3A%2F%2Fwww.linuxvirtualserver.org%2Fzh%2Flvs4.html)
 NAT模式安装详解   [http://www.cnblogs.com/liwei0526vip/p/6370103.html](https://link.jianshu.com?t=http%3A%2F%2Fwww.cnblogs.com%2Fliwei0526vip%2Fp%2F6370103.html)
 开发眼光看lvs      [http://blog.hesey.net/2013/02/introduce-to-load-balance-and-lvs-briefly.html](https://link.jianshu.com?t=http%3A%2F%2Fblog.hesey.net%2F2013%2F02%2Fintroduce-to-load-balance-and-lvs-briefly.html)
 lvs 介绍      [http://www.it165.net/admin/html/201401/2248.html](https://link.jianshu.com?t=http%3A%2F%2Fwww.it165.net%2Fadmin%2Fhtml%2F201401%2F2248.html)

