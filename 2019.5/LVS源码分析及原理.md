LVS英文网站：http://www.linuxvirtualserver.org/

LVS中文网站：http://www.linuxvirtualserver.org/zh/index.html

LVS论坛：http://zh.linuxvirtualserver.org/

原文摘自 http://zh.linuxvirtualserver.org/blog/3309



# 为什么是LVS

ipvs的3种技术：

1)VS/NAT机制，第一反应是，netfitler已经实现nat的功能，ipvs不会再自己实现一边相同的功能吧!?!

2)VS/TUN机制，应该和IPsec VPN的实现机制差不多吧。

3)VS/DR机制，以前做集群方案的时候听XX说过，原来起源于ipvs

这里有两点值得高兴.
1)LVS是中国人搞的，开源社区很少看到中国人。
2)ipvs是基于netfilter机制的。娃哈哈。



# LVS模块组织

## ipvs相关代码

include/linux/ip_vs.h

include/net/ip_vs.h

net/netfilter/ipvs/目录下共23个文件。



## 从Makefile分析23个文件

http://www.linuxvirtualserver.org/zh/index.html文中提到的各种连接调度算法。每一种算法
分别独立实现为一个内核模块：ip_vs_rr.o,ip_vs_wrr.o,ip_vs_lc.o,ip_vs_wlc.o,ip_vs_lblc.o,ip_vs_lblcr.o,ip_vs_dh.o,ip_vs_sh.o,ip_vs_sed.o,ip_vs_nq.o.共对应了10个C文件。
每一个C文件主要实现属于自己调度算法的struct ip_vs_scheduler结构,在模块初始化的时候注册到ipvs系统中，
在模块卸载的时候，从ipvs系统中注销掉自己。



ipvs目前支持的传输层协议tcp，udp，ah，esp四种，tcp，udp很常见，ah，esp属于IPsec协议族的协议。相信了解IPsec VPN的人对这很熟悉。其中对应的ip_vs_proto_udp.c,ip_vs_proto_tcp.c,ip_vs_proto_ah_esp.c共3个C文件。
主要实现各自协议的struct ip_vs_protocol数据结构。

连接调度算法每一个种算法都设计为独立内核模块，可以在运行时动态的按需加载。然而对传输层协议的支持没有设计为独立的内核模块，不可以运行时决定支持那种协议。而是在编译的时候决定的。编译的时候配置了支持某种传输层协议。在Makefile中会把对应的.o静态连接到ip_vs.o中:
ip_vs_proto-objs-y :=
ip_vs_proto-objs-$(CONFIG_IP_VS_PROTO_TCP) += ip_vs_proto_tcp.o
ip_vs_proto-objs-$(CONFIG_IP_VS_PROTO_UDP) += ip_vs_proto_udp.o
ip_vs_proto-objs-$(CONFIG_IP_VS_PROTO_AH_ESP) += ip_vs_proto_ah_esp.o
并且在ip_vs_proto.c文件
ip_vs_protocol_init函数中把对应的传输层协议注册到ip_vs系统中:
#ifdef CONFIG_IP_VS_PROTO_TCP
REGISTER_PROTOCOL(&ip_vs_protocol_tcp);
#endif
#ifdef CONFIG_IP_VS_PROTO_UDP
REGISTER_PROTOCOL(&ip_vs_protocol_udp);
#endif
#ifdef CONFIG_IP_VS_PROTO_AH
REGISTER_PROTOCOL(&ip_vs_protocol_ah);
#endif
#ifdef CONFIG_IP_VS_PROTO_ESP
REGISTER_PROTOCOL(&ip_vs_protocol_esp);
#endif

ip_vs_ftp.o是ip_vs的应用helper，对应ip_vs_ftp.c.由ftp协议猜想可能是用来解决关联连接的问题。如果是为了解决关联连接问题，这里可以看到ipvs并不完善，其实还有很多工作需要做。比如netfilter中处理关联连接除了支持ftp外，已经支持了h323，pptp等其他很多协议。

剩下9个文件是ip_vs.o的必选文件。我这里为了与以上各部门区分，称他们为ip_vs_core.下面主要的精力会集中分析ip_vs_core.



# 源码分析-netfilter hook

内核版本linux-2.6.32.2
ipvs是基于netfilter框架的，在这里我们先了解一下ipvs的hook在netfilter系统中的位置。
![](assets\linux2.6.32.2netfilter.JPG)
上图中红色标注的hook是ip_vs注册的。
长期工作在linux2.4.35下，对于图中nf_defrag_ipv4.o, iptable_raw.o,iptable_security.o,selinux.o很陌生。



```
  static struct nf_hook_ops ip_vs_ops[] __read_mostly = {
  /* After packet filtering, forward packet through VS/DR, VS/TUN,
  
  - or VS/NAT(change destination), so that filtering rules can be
  - applied to IPVS. */
    /××
    ×	由上图可见ip_vs的hook是挂在iptable_filter.o模块的钩子后面。
    ×	因此应用层iptables对应的-t filter表的规则，可以和ip_vs.o一起工作。
    ×	经过LOCAL_IN的连接，也就是需要转发到后端真实服务器的链接。
    × 因此这里ip_vs.o不会把数据包按协议栈的流程发送到传输层，
    × 而是根据当前的模式VS/DR或VS/TUN或VS/NAT，经过对应的处理，把包转发给后端真实的服务器。
    × 由下面ip_vs_out的钩子可以猜到这个函数在VS/NAT模式下需要实现dnat功能。
    ×/
    {
    .hook	= ip_vs_in,
    .owner	= THIS_MODULE,
    .pf	= PF_INET,
    .hooknum = NF_INET_LOCAL_IN,
    .priority = 100,
    },
    /* After packet filtering, change source only for VS/NAT */
    /××
    ×	由上图可见ip_vs的hook是挂在iptable_filter.o模块的钩子后面。
    ×	因此应用层iptables对应的-t filter表的规则，可以和ip_vs.o一起工作。
    ×	根据“change source only for VS/NAT”猜想，这个函数的功能是在VS/NAT
    ×	模式下，处理真实服务器——>客户端方向的回包，把回包中的源IP，从真实
    × 服务器的IP修改为本机（LB）的IP。
    ×	这里可见判断，NAT实现ip_vs并没有使用netfilter的nat，而是自己实现
    × 了一套。
    ×/
    {
    .hook	= ip_vs_out,
    .owner	= THIS_MODULE,
    .pf	= PF_INET,
    .hooknum = NF_INET_FORWARD,
    .priority = 100,
    },
    /* After packet filtering (but before ip_vs_out_icmp), catch icmp
  - destined for 0.0.0.0/0, which is for incoming IPVS connections */
    /**
  - 猜想做client和真实服务器的ICMP中转。
    */
    {
    .hook	= ip_vs_forward_icmp,
    .owner	= THIS_MODULE,
    .pf	= PF_INET,
    .hooknum = NF_INET_FORWARD,
    .priority = 99,
    },
    /* Before the netfilter connection tracking, exit from POST_ROUTING */
    {
    .hook	= ip_vs_post_routing,
    .owner	= THIS_MODULE,
    .pf	= PF_INET,
    .hooknum = NF_INET_POST_ROUTING,
    .priority = NF_IP_PRI_NAT_SRC-1,
    },
    };
  
  /*
  
  - It is hooked before NF_IP_PRI_NAT_SRC at the NF_INET_POST_ROUTING
  - chain, and is used for VS/NAT.
  - It detects packets for VS/NAT connections and sends the packets
  - immediately. This can avoid that iptable_nat mangles the packets
  - for VS/NAT.
    */
    /××
    ×	由上图可以看到这个钩子是在iptable_nat,selinux.o,ip_conntrack.o模块的钩子之前。
    ×	设置了skb->ipvs_property这个标志的包，将返回NF_STOP，不会继续跑后面的钩子。
    ×	ip_vs是不能与selinux.o协同工作，启动ip_vs.o后，属于ip_vs的包将不会
    ×	经过selinux hook处理，导致功能失效。
    ×	至于iptables_nat, 不经过ip_vs处理的连接，还是可以正常使用iptables -t nat下发的规则。
    ×	经过ip_vs的包，如果iptables的nat表有对应的规则，那末这个规则将失效。对于经过ip_vs的
    ×	自己实现的nat。也就是说当一个连接同时符合iptables -nat规则和ip_vs规则的时候，iptables
  - nat规则失效，ip_vs正常工作。
    ×/
    static unsigned int ip_vs_post_routing(unsigned int hooknum,
    struct sk_buff *skb,
    const struct net_device *in,
    const struct net_device *out,
    int (*okfn)(struct sk_buff *))
    {
    if (!skb->ipvs_property)
    return NF_ACCEPT;
    /* The packet was sent from IPVS, exit this chain */
    return NF_STOP;
    }
```

  

这里有一个问题:
在LOCAL_IN,优先级为100的钩子钩子有两个ip_vs_in,nf_nat_fn,到底谁在前谁在后?
这种情况，取决于注册钩子的顺序，也是内核模块加载顺序。

既然这样，为什么上图把ip_vs_in画在前，nf_nat_fn画在后?
由于我猜想ip_vs_in要部分接管nat功能，只有ip_vs_in位于nf_nat_fn前面才能实现这个功能。

只有ip_vs_in位于nf_nat_fn前面才能实现这个功能，那就意味这必须先加载ip_vs.o,后加载
iptable_nat.o模块?
对，由于netfilter实现先注册的在前面：
int nf_register_hook(struct nf_hook_ops *reg)
{
struct nf_hook_ops *elem;
int err;

err = mutex_lock_interruptible(&nf_hook_mutex);
if (err < 0)
return err;
list_for_each_entry(elem, &nf_hooks[reg->pf][reg->hooknum], list) {
if (reg->priority < elem->priority)
break;
}
list_add_rcu(&reg->list, elem->list.prev);
mutex_unlock(&nf_hook_mutex);
return 0;
}

这里个人的看法是设序设计当中应该尽量避免二义性，减少依赖性。
建议ip_vs_in的优先改为(100 > x > 50)范围，最好为75。
最佳的方案最好是能和netfilter项目组联系，在下面添加一个属于自己的优先级。不过可能2个不同的开源项目操作起来不方便。
enum nf_ip_hook_priorities {
NF_IP_PRI_FIRST = INT_MIN,
NF_IP_PRI_CONNTRACK_DEFRAG = -400,
NF_IP_PRI_RAW = -300,
NF_IP_PRI_SELINUX_FIRST = -225,
NF_IP_PRI_CONNTRACK = -200,
NF_IP_PRI_MANGLE = -150,
NF_IP_PRI_NAT_DST = -100,
NF_IP_PRI_FILTER = 0,
NF_IP_PRI_SECURITY = 50,
NF_IP_PRI_NAT_SRC = 100,
NF_IP_PRI_SELINUX_LAST = 225,
NF_IP_PRI_CONNTRACK_CONFIRM = INT_MAX,
NF_IP_PRI_LAST = INT_MAX,
};



# 小插曲：谁抢谁的饭碗

![](assets\ipvs_pk_netfilter_0.JPG)

内核版本linux-2.6.32.2，今天发现这个版本的netfilter有一个CLUSTERIP的target。看样子netfilter要发力负载均衡。ipvs做了netfilter的NAT。netfilter又在做ipvs的负载均衡。社会有些不太和谐，呵呵。