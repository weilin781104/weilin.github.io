LVS DR模式试验时发现的问题：由于vip设置错误引起arp不响应进而无法访问

| 服务器名 | 内网地址       | 外网地址    | os版本                               |
| -------- | -------------- | ----------- | ------------------------------------ |
| Web01    | 192.168.11.123 | 10.18.14.72 | CentOS Linux release 7.6.1810 (Core) |
| Web02    | 192.168.11.124 | 10.18.14.80 | CentOS Linux release 7.6.1810 (Core) |

从Web02能正常ping 192.168.11.123

然后再web01的lo上配置vip 192.168.11.129，发现连192.168.11.123都ping不通了。

```
[root@web01 network-scripts]# cat ifcfg-lo:0
DEVICE=lo:0
IPADDR=192.168.11.129
NETMASK=255.255.255.0
#BROADCAST=127.255.255.255
ONBOOT=yes
#NAME=loopback
```

[root@web01 network-scripts]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet 192.168.11.129/24 brd 192.168.11.255 scope global lo:0
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:57:02:a1 brd ff:ff:ff:ff:ff:ff
    inet 192.168.11.123/24 brd 192.168.11.255 scope global ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fe57:2a1/64 scope link 
       valid_lft forever preferred_lft forever

[root@web02 ~]# ping 192.168.11.123
PING 192.168.11.123 (192.168.11.123) 56(84) bytes of data.
From 192.168.11.124 icmp_seq=1 Destination Host Unreachable
From 192.168.11.124 icmp_seq=2 Destination Host Unreachable
From 192.168.11.124 icmp_seq=3 Destination Host Unreachable
From 192.168.11.124 icmp_seq=4 Destination Host Unreachable



排查问题发现：

web02没有192.168.11.123的arp信息

```
[root@web02 ~]# arp -n -i ens33
Address                  HWtype  HWaddress           Flags Mask            Iface
192.168.11.129                   (incomplete)                              ens33
192.168.11.52            ether   14:18:77:70:c3:09   C                     ens33
192.168.11.123                   (incomplete)                              ens33
192.168.11.130                   (incomplete)                              ens33
192.168.11.89            ether   00:0c:29:34:84:4a   C                     ens33
192.168.11.1             ether   00:1b:53:4b:00:3f   C                     ens33
```

因为web01 对于web02发出arp请求（who has 192.168.11.123）不响应

```
[root@web01 ~]# tcpdump -i any -n -vv|grep ARP|grep 192.168.11.123
tcpdump: listening on any, link-type LINUX_SLL (Linux cooked), capture size 262144 bytes
14:29:49.837649 ARP, Ethernet (len 6), IPv4 (len 4), Request who-has 192.168.11.123 tell 192.168.11.124, length 46
14:29:50.838960 ARP, Ethernet (len 6), IPv4 (len 4), Request who-has 192.168.11.123 tell 192.168.11.124, length 46
14:29:51.840677 ARP, Ethernet (len 6), IPv4 (len 4), Request who-has 192.168.11.123 tell 192.168.11.124, length 46
14:29:53.839478 ARP, Ethernet (len 6), IPv4 (len 4), Request who-has 192.168.11.123 tell 192.168.11.124, length 46
14:29:54.840567 ARP, Ethernet (len 6), IPv4 (len 4), Request who-has 192.168.11.123 tell 192.168.11.124, length 46
```

根本原因是vip的广播地址设置错了，把vip删除就能应答arp了

```
14:32:49.922297 ARP, Ethernet (len 6), IPv4 (len 4), Request who-has 192.168.11.123 tell 192.168.11.124, length 46
14:32:50.924816 ARP, Ethernet (len 6), IPv4 (len 4), Request who-has 192.168.11.123 tell 192.168.11.124, length 46
14:32:50.924859 ARP, Ethernet (len 6), IPv4 (len 4), Reply 192.168.11.123 is-at 00:0c:29:57:02:a1, length 28
14:32:55.930881 ARP, Ethernet (len 6), IPv4 (len 4), Request who-has 192.168.11.124 tell 192.168.11.123, length 28
```



vip设置时需要把组播地址设置成vip即可

```
[root@web01 network-scripts]# cat ifcfg-lo:0
DEVICE=lo:0
IPADDR=192.168.11.129
NETMASK=255.255.255.255
#NETWORK=127.0.0.0
#BROADCAST=127.255.255.255
ONBOOT=yes
#NAME=loopback

[root@web01 network-scripts]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet 192.168.11.129/32 brd 192.168.11.129 scope global lo:0
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:57:02:a1 brd ff:ff:ff:ff:ff:ff
    inet 192.168.11.123/24 brd 192.168.11.255 scope global ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fe57:2a1/64 scope link 
       valid_lft forever preferred_lft forever


```

