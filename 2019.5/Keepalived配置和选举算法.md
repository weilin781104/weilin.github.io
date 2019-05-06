# Keepalived

## 配置

keepalived的配置位于/etc/keepalived/keepalived.conf，配置文件格式包含多个必填/可选的配置段，部分重要配置含义如下：

- global_defs: 全局定义块，定义主从切换时通知邮件的SMTP配置。
- vrrp_instance: vrrp实例配置。
- vrrp_script: 健康检查脚本配置。

细分下去，vrrp_instance配置段包括：

- state: 实例角色。分为一个MASTER和一(多)个BACKUP。
- virtual_router_id: 标识该虚拟路由器的ID，有效范围为0-255。
- priority: 优先级初始值，竞选MASTER用到，有效范围为0-255。
- advert_int: VRRP协议通告间隔。
- interface: VIP所绑定的网卡，指定处理VRRP多播协议包的网卡。
- mcast_src_ip: 指定发送VRRP协议通告的本机IP地址。
- authentication: 认证方式。
- virtual_ipaddress: VIP。
- track_script: 健康检查脚本。

vrrp_script配置段包括：

- script: 一句指令或者一个脚本文件，需返回0(成功)或非0(失败)，keepalived以此为依据判断其监控的服务状态。
- interval: 健康检查周期。
- weight: 优先级变化幅度。
- fall: 判定服务异常的检查次数。
- rise: 判定服务正常的检查次数。

## 选举算法

keepalived中优先级高的节点为MASTER。MASTER其中一个职责就是响应VIP的arp包，将VIP和mac地址映射关系告诉局域网内其他主机，同时，它还会以多播的形式（目的地址224.0.0.18）向局域网中发送VRRP通告，告知自己的优先级。网络中的所有BACKUP节点只负责处理MASTER发出的多播包，当发现MASTER的优先级没自己高，或者没收到MASTER的VRRP通告时，BACKUP将自己切换到MASTER状态，然后做MASTER该做的事：1.响应arp包，2.发送VRRP通告。

MASTER和BACKUP节点的优先级如何调整？

首先，每个节点有一个初始优先级，由配置文件中的priority配置项指定，MASTER节点的priority应比BAKCUP高。运行过程中keepalived根据vrrp_script的weight设定，增加或减小节点优先级。规则如下：

\1. 当weight > 0时，vrrp_script script脚本执行返回0(成功)时优先级为priority + weight, 否则为priority。当BACKUP发现自己的优先级大于MASTER通告的优先级时，进行主从切换。

\2. 当weight < 0时，vrrp_script script脚本执行返回非0(失败)时优先级为priority + weight, 否则为priority。当BACKUP发现自己的优先级大于MASTER通告的优先级时，进行主从切换。

\3. 当两个节点的优先级相同时，以节点发送VRRP通告的IP作为比较对象，IP较大者为MASTER。

主从的优先级初始值priority和变化量weight设置非常关键，配错的话会导致无法进行主从切换。比如，当MASTER初始值定得太高，即使script脚本执行失败，也比BACKUP的priority + weight大，就没法进行VIP漂移了。所以priority和weight值的设定应遵循: abs(MASTER priority – BAKCUP priority) < abs(weight)。

另外，当网络中不支持多播(例如某些云环境)，或者出现网络分区的情况，keepalived BACKUP节点收不到MASTER的VRRP通告，就会出现脑裂(split brain)现象，此时集群中会存在多个MASTER节点。没有一致性算法，好像脑裂问题无解。



以下几种情况发生主备切换：

1. master主机或网络故障，不发送VRRP通告，即backup收不到VRRP通告

2. master健康检查脚本失败，引起优先级降级，即backup收到优先级小于自己的VRRP通告

3. 原master健康检查脚本成功，优先级恢复，即backup收到优先级大于自己的VRRP通告

4. 如果未设置weight时，weight默认值为0，此时当vrrp_script连续检测失败时，vrrp实例进入FAULT状态。会导致VIP转移，原来如此！！