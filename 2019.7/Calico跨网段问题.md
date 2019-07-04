# Calico 简介

Calico 是一个基于BGP协议的网络互联解决方案。它是一个纯3层的方法，使用路由来实现报文寻址和传输。 
相比 flannel, ovs等SDN解决方案，Calico 避免了层叠网络带来的性能损耗。将节点当做 router ，位于节点上的 container 被当做 router 的直连设备。利用 Kernel 来实现高效的路由转发。 节点间的路由信息通过 BGP 协议在整个 Calico 网络中传播。 具有以下特点： 

1. 在 calico 中的数据包不需要进行封包和解封。 
2. 基于三层网络通信，troubleshoot 会更方便。 
3. 网络安全策略使用 ACL 定义，基于 iptables 实现，比起 overlay 方案中的复杂机制更只管和容易操作。

# Environment

![](assets\20180326111303747.png)

| server    | ip              | mac               | gw mac            |
| --------- | --------------- | ----------------- | ----------------- |
| walker-1  | 172.16.6.47     | fa:16:3e:02:8b:17 | 00:23:89:8C:E8:31 |
| walker-2  | 172.16.6.249    | fa:16:3e:8c:21:13 | 00:23:89:8C:E8:31 |
| demi-1    | 172.16.199.114  | fa:16:3e:d9:a0:5e | 00:23:89:8C:E8:31 |
| busybox-1 | 192.168.187.211 | 3a:1d:1e:91:f5:9e | 66:39:fa:e7:9f:a9 |
| busybox-2 | 192.168.135.74  | de:16:fc:1c:44:35 | 5a:4a:df:5e:c9:6c |
| busybox-3 | 192.168.121.2   | de:16:fc:1c:44:35 | 8e:6b:fa:f7:5d:3b |

node2 单独位于vlan 199 中,和master以及node1间的通信需要经过网关（Router）转发。

# 使用 IP-in-IP

## ipip enable=false

```
$ calicoctl apply -f - << EOF
apiVersion: v1
kind: ipPool
metadata:
  cidr: 192.168.0.0/16
spec:
  nat-outgoing: true
EOF
```


ipip 模式禁用时，busybox-3 和 busybox-{1,2} 之间无法通信。

分析如下：

主机路由：

```
[root@walker-1 manifests]# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         172.16.6.254    0.0.0.0         UG    0      0        0 eth0
169.254.169.254 172.16.6.87     255.255.255.255 UGH   0      0        0 eth0
172.16.6.0      0.0.0.0         255.255.255.0   U     0      0        0 eth0
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
192.168.121.0   172.16.199.114  255.255.255.192 UG    0      0        0 eth0
192.168.135.64  172.16.6.249    255.255.255.192 UG    0      0        0 eth0
192.168.187.192 0.0.0.0         255.255.255.192 U     0      0        0 *
192.168.187.209 0.0.0.0         255.255.255.255 UH    0      0        0 calic6611247c43
192.168.187.211 0.0.0.0         255.255.255.255 UH    0      0        0 calie50081a277c
```


从busybox-1 发往 busybox-3 的报文头部如下所示：

| src max           | dst mac           | src ip          | dst ip        |
| ----------------- | ----------------- | --------------- | ------------- |
| 3a:1d:1e:91:f5:9e | 66:39:fa:e7:9f:a9 | 192.168.187.211 | 192.168.121.2 |

根据宿主机路由，报文会从eth0 发往 172.16.199.114。

由于二者位于不通广播域，需要通过网关转发。因此报文的 dst mac 会被修改为 172.16.6.254(gw) 对应的mac。

| src max           | dst mac           | src ip          | dst ip        | enc src IP | enc dst IP |
| ----------------- | ----------------- | --------------- | ------------- | ---------- | ---------- |
| fa:16:3e:02:8b:17 | 00:23:89:8C:E8:31 | 192.168.187.211 | 192.168.121.2 |            |            |

gw 收到该报文后，会比对本地路由条目。由于 router 中并没有到 192.168.121.0\26 网段的路由，因此报文被丢弃。

## ipip enable=true

```
$ calicoctl apply -f - << EOF
apiVersion: v1
kind: ipPool
metadata:
  cidr: 192.168.0.0/16
spec:
  ipip:
    enabled: true
    mode: always
  nat-outgoing: true
EOF
```


这种模式下，可实现跨网段节点上容器的互通。

```
[root@walker-1 manifests]# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         172.16.6.254    0.0.0.0         UG    0      0        0 eth0
169.254.169.254 172.16.6.87     255.255.255.255 UGH   0      0        0 eth0
172.16.6.0      0.0.0.0         255.255.255.0   U     0      0        0 eth0
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
192.168.121.0   172.16.199.114  255.255.255.192 UG    0      0        0 tunl0
192.168.135.64  172.16.6.249    255.255.255.192 UG    0      0        0 tunl0
192.168.187.192 0.0.0.0         255.255.255.192 U     0      0        0 *
192.168.187.209 0.0.0.0         255.255.255.255 UH    0      0        0 calic6611247c43
192.168.187.211 0.0.0.0         255.255.255.255 UH    0      0        0 calie50081a277c
```


busybox-1 发送报文至 busybox-3 时，根据 master 上路由，会经过 tunl0 设备。tunl0 会为原来的IP报文在封装一层IP协议头。过程如下：

busybox-1 向 busybox-3 发送 icmp 报文（略去容器至 calico 网卡步骤）。

| src mac           | dst mac           | src ip          | dst ip        |
| ----------------- | ----------------- | --------------- | ------------- |
| 3a:1d:1e:91:f5:9e | 66:39:fa:e7:9f:a9 | 192.168.187.211 | 192.168.121.2 |


报文被node 截获，查询本机路由后由 tunl0 设备将报文发出。

| src mac           | dst mac           | src ip          | dst ip        | enc src ip  | enc dst ip     |
| ----------------- | ----------------- | --------------- | ------------- | ----------- | -------------- |
| fa:16:3e:02:8b:17 | 00:23:89:8C:E8:31 | 192.168.187.211 | 192.168.121.2 | 172.16.6.47 | 172.16.199.114 |


报文发送至网关（router), 根据目的IP将报文路由至 172.16.199.114（略去ARP等步骤）。

|src max	|dst mac	|src ip	|dst ip	|enc src IP	|enc dst IP|
| ----------------- | ----------------- | --------------- | ------------- | ----------- | -------------- |
|00:23:89:8C:E8:31	|fa:16:3e:d9:a0:5e	|192.168.187.211	|192.168.121.2|	172.16.6.47	|172.16.199.114|
到达 demi-1 后，根据 demi-1 上的路由策略，将报文发送至 busybox-3（略去容器至 calico 网卡步骤） 。

|src max	|dst mac	|src ip	|dst ip	|
| ----------------- | ----------------- | --------------- | ------------- |
|8e:6b:fa:f7:5d:3b	|de:16:fc:1c:44:35	|192.168.187.211	|192.168.121.2|
Note: ==容器的 gw 为和该容器对应的 calico 网卡。==

虽然实现了 calico 跨网段通信，但对于 busybox-{1,2} 间的通信来说，IP-in-IP 就有点多余了，因为2者宿主机处于同一广播域，2层互通，直接走主机路由即可。

## calico cross-subnet

```
$ calicoctl apply -f - << EOF
apiVersion: v1
kind: ipPool
metadata:
  cidr: 192.168.0.0/16
spec:
  ipip:
    enabled: true
    mode: cross-subnet
  nat-outgoing: true
EOF
```


为了解决IP-in-IP 模式下，同网段封装报文的问题，calico 提供了 cross-subnet 的配置选项

```
[root@walker-1 k8s]# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         172.16.6.254    0.0.0.0         UG    0      0        0 eth0
169.254.169.254 172.16.6.87     255.255.255.255 UGH   0      0        0 eth0
172.16.6.0      0.0.0.0         255.255.255.0   U     0      0        0 eth0
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
192.168.121.0   172.16.199.114  255.255.255.192 UG    0      0        0 tunl0
192.168.135.64  172.16.6.249    255.255.255.192 UG    0      0        0 eth0
192.168.187.192 0.0.0.0         255.255.255.192 U     0      0        0 *
192.168.187.209 0.0.0.0         255.255.255.255 UH    0      0        0 calic6611247c43
192.168.187.211 0.0.0.0         255.255.255.255 UH    0      0        0 calie50081a277c
```


从主机路由可看出，对于同一网段中的路由，直接走 eth0 网卡。

默认情况下，calico 在启动时会自动选择一块网卡。当主机上有多块网卡时，为了保证路由的正确性，需要手动指定 calico 使用哪块物理网卡。参考一下链接：

http://docs.projectcalico.org/v2.3/usage/configuration/node

Note

```
The cross-subnet mode option requires each Calico node to be configured with the IP address and subnet of the host. However, the subnet configuration was only introduced in Calico v2.1. If any nodes in your deployment were originally created with an older version of Calico, or if you if you are unsure whether your deployment is configured correctly, follow the steps in Upgrading from pre-v2.1 before enabling cross-subnet IPIP.
```

