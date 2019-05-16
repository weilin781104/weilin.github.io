# 背景

目前项目中采用codis方案的redis集群，来提高redis集群的容量和吞吐量。

但是对于数据量较小，并且读多写少的项目，可以用一主多从（主从复制）和sentinel自动切换来构建，目前服务器内存都比较大，应该能满足数据容量，然后一主多从多台服务器都可以读数据，如果要负载均衡读数据，可以在前面放两台keepalived做loadbalance。 sentinel用来做故障转移。 为了简化客户端开发，为master节点设置一个vip，sentinel故障转移时触发脚本进行vip的漂移。 这样客户端只需要用标准客户端读请求连接keepalived vip，写请求连接master的vip。



# 网络拓扑和设备信息



## 网络拓扑

![](assets\20190515153758.png)

## 设备信息

| 服务器名 | 内网地址       | 外网地址     | 应用       | os版本                 |
| -------- | -------------- | ------------ | ---------- | ---------------------- |
| LB01     | 192.168.11.121 | 10.18.14.54  | Keepalived | CentOS 7.6.1810 (Core) |
| LB02     | 192.168.11.122 | 10.18.14.64  | Keepalived | CentOS 7.6.1810 (Core) |
| Redis01  | 192.168.11.123 | 10.18.14.72  | Redis      | CentOS 7.6.1810 (Core) |
| Redis02  | 192.168.11.124 | 10.18.13.166 | Redis      | CentOS 7.6.1810 (Core) |
| Redis03  | 192.168.11.125 | 10.18.14.80  | Redis      | CentOS 7.6.1810 (Core) |
| Client   | 192.168.11.89  | 10.18.14.88  | Redis      | CentOS 7.3.1611(Core)  |



# 部署

## 环境准备

[参见软四层实现部署](../2019.4/软四层实现部署.md)

在软四层实现部署的基础上增加一台web03， web01|web02|web03三台作为redis节点和sentinel节点。



## 安装redis节点



### 一主两从

#### web01(redis master)

```
[root@web01 ~]# cd /usr
[root@web01 usr]# curl http://download.redis.io/releases/redis-5.0.4.tar.gz -o redis-5.0.4.tar.gz
[root@web01 usr]# tar xzf redis-5.0.4.tar.gz
[root@web01 usr]# ln -s redis-5.0.4 redis
[root@web01 usr]# cd redis
[root@web01 redis]# make
[root@web01 redis]# mkdir bin
[root@web01 redis]# cp src/redis-server ./bin
[root@web01 redis]# cp src/redis-sentinel ./bin
[root@web01 redis]# cp src/redis-cli ./bin  
[root@web01 redis]# mkdir conf
[root@web01 redis]# cp *.conf conf/
[root@web01 redis]# cd conf
[root@web01 conf]# vi redis.conf
#bind 127.0.0.1
protected-mode no
daemonize yes
logfile "/var/log/redis.log"
masterauth 123456
requirepass 123456
maxmemory 1g
maxmemory-policy volatile-lru
appendonly yes

[root@web01 conf]# cd ..
[root@web01 redis]# ./bin/redis-server  ./conf/redis.conf

[root@web01 redis]# ss -tunlp|grep 6379
tcp    LISTEN     0      128       *:6379                  *:*                   users:(("redis-server",pid=11231,fd=7))
tcp    LISTEN     0      128      :::6379                 :::*                   users:(("redis-server",pid=11231,fd=6))

[root@web01 redis]# ./bin/redis-cli -a 123456 info


```

#### web02(redis slave)

```
[root@web02 ~]# cd /usr
[root@web02 usr]# scp 192.168.11.123:/usr/redis-5.0.4.tar.gz ./
[root@web02 usr]# tar xzf redis-5.0.4.tar.gz
[root@web02 usr]# ln -s redis-5.0.4 redis
[root@web02 usr]# cd redis
[root@web02 redis]# make
[root@web02 redis]# mkdir bin
[root@web02 redis]# cp src/redis-server ./bin
[root@web02 redis]# cp src/redis-sentinel ./bin
[root@web02 redis]# cp src/redis-cli ./bin 
[root@web02 redis]#  mkdir conf
[root@web02 redis]# scp 192.168.11.123:/usr/redis/conf/redis.conf ./conf/
[root@web02 redis]# vi conf/redis.conf
replicaof 192.168.11.123 6379

[root@web02 redis]# ./bin/redis-server  ./conf/redis.conf
[root@web02 redis]# ss -tunlp|grep 6379
tcp    LISTEN     0      128       *:6379                  *:*                   users:(("redis-server",pid=11180,fd=7))
tcp    LISTEN     0      128      :::6379                 :::*                   users:(("redis-server",pid=11180,fd=6))

[root@web02 redis]#  ./bin/redis-cli -a 123456 info 

```

#### web03(redis slave)

```
[root@web03 ~]# cd /usr
[root@web02 usr]# scp 192.168.11.123:/usr/redis-5.0.4.tar.gz ./
[root@web03 usr]# tar xzf redis-5.0.4.tar.gz
[root@web03 usr]#  ln -s redis-5.0.4 redis
[root@web03 usr]# cd redis
[root@web03 redis]# make
[root@web03 redis]# mkdir bin
[root@web03 redis]# cp src/redis-server ./bin
[root@web03 redis]# cp src/redis-sentinel ./bin
[root@web03 redis]# cp src/redis-cli ./bin 
[root@web03 redis]# mkdir conf
[root@web03 redis]# scp 192.168.11.124:/usr/redis/conf/redis.conf ./conf/ 
[root@web03 redis]# ./bin/redis-server  ./conf/redis.conf
[root@web03 redis]#  ss -tunlp|grep 6379
tcp    LISTEN     0      128       *:6379                  *:*                   users:(("redis-server",pid=11800,fd=7))
tcp    LISTEN     0      128      :::6379                 :::*                   users:(("redis-server",pid=11800,fd=6))
[root@web03 redis]# ./bin/redis-cli -a 123456 info
```



#### 测试主从复制功能

```
[root@devhost log]# redis-cli -h 192.168.11.123 -p 6379 -a 123456  
192.168.11.123:6379> set test:username weilin
OK
192.168.11.123:6379> get test:username
"weilin"
192.168.11.123:6379> quit

[root@devhost log]# redis-cli -h 192.168.11.124 -p 6379 -a 123456  
192.168.11.124:6379> get test:username
"weilin"
192.168.11.124:6379> quit

[root@devhost log]# redis-cli -h 192.168.11.125 -p 6379 -a 123456  
192.168.11.125:6379> get test:username
"weilin"
192.168.11.125:6379> quit

```



### Sentinel

#### web01(sentinel1)

```
[root@web01 redis]# cd /var/lib
[root@web01 lib]# mkdir redis
[root@web01 lib]# cd redis
[root@web01 redis]# vi reconfig_master.sh
#!/bin/bash
MASTER_IP=$6
LOCAL_IP='192.168.11.123'
VIP='10.18.14.94'
NETMASK='20'
INTERFACE='ens34'
if [ ${MASTER_IP} = ${LOCAL_IP} ]; then
  ip addr add ${VIP}/${NETMASK} dev ${INTERFACE}
  arping -q -c 3 -A ${VIP} -I ${INTERFACE}
  exit 0
else
  ip addr del ${VIP}/${NETMASK} dev ${INTERFACE}
  exit 0
fi
exit 1

[root@web01 redis]# chmod u+x /var/lib/redis/reconfig_master.sh
[root@web01 redis]# ip addr add 10.18.14.94/20 dev ens34
[root@web01 redis]# arping -q -c 3 -A 10.18.14.94 -I ens34
[root@web01 redis]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:57:02:a1 brd ff:ff:ff:ff:ff:ff
    inet 192.168.11.123/24 brd 192.168.11.255 scope global ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fe57:2a1/64 scope link 
       valid_lft forever preferred_lft forever
3: ens34: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:57:02:ab brd ff:ff:ff:ff:ff:ff
    inet 10.18.14.72/20 brd 10.18.15.255 scope global dynamic ens34
       valid_lft 25679sec preferred_lft 25679sec
    inet 10.18.14.94/20 scope global secondary ens34
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fe57:2ab/64 scope link 
       valid_lft forever preferred_lft forever


[root@web01 redis]# cd /usr/redis/conf
[root@web01 conf]# vi sentinel.conf
protected-mode no
daemonize yes
logfile "/var/log/sentinel.log"
sentinel monitor master1 192.168.11.123 6379 2
sentinel auth-pass master1 123456
sentinel down-after-milliseconds mymaster 5000
sentinel parallel-syncs master1 1
sentinel failover-timeout master1 30000
sentinel client-reconfig-script master1 /var/lib/redis/reconfig_master.sh

[root@web01 redis]# ./bin/redis-server ./conf/sentinel.conf  --sentinel
[root@web01 redis]# ./bin/redis-cli -p 26379  info
```



#### web02(sentinel2)

```
[root@web02 redis]#  mkdir -p /var/lib/redis
[root@web02 redis]# scp 192.168.11.123:/var/lib/redis/reconfig_master.sh /var/lib/redis/
[root@web02 redis]# vi /var/lib/redis/reconfig_master.sh
LOCAL_IP='192.168.11.124'

[root@web02 redis]# chmod u+x /var/lib/redis/reconfig_master.sh

[root@web02 redis]# scp 192.168.11.123:/usr/redis/conf/sentinel.conf /usr/redis/conf/
[root@web02 redis]# ./bin/redis-server ./conf/sentinel.conf  --sentinel
[root@web02 redis]# ./bin/redis-cli -p 26379 info

```



#### web03(sentinel3)

```
[root@web03 redis]#  mkdir -p /var/lib/redis
[root@web03 redis]# scp 192.168.11.123:/var/lib/redis/reconfig_master.sh /var/lib/redis/
[root@web03 redis]# vi /var/lib/redis/reconfig_master.sh
LOCAL_IP='192.168.11.125'

[root@web03 redis]# chmod u+x /var/lib/redis/reconfig_master.sh

[root@web03 redis]# scp 192.168.11.123:/usr/redis/conf/sentinel.conf /usr/redis/conf/
[root@web03 redis]# ./bin/redis-server ./conf/sentinel.conf  --sentinel
[root@web03 redis]# ./bin/redis-cli -p 26379  info
```



#### 故障转移测试



```
#客户端通过writevip连到redis master
[root@devhost codis]# redis-cli -h 10.18.14.94 -a 123456
10.18.14.94:6379> get test:username
"weilin"

#客户端直接连到salve redis上
[root@devhost log]# redis-cli -h 192.168.11.124  -a 123456         
192.168.11.124:6379> get test:username
"weilin"

[root@devhost ~]# redis-cli -h 192.168.11.125  -a 123456  
192.168.11.125:6379> get test:username
"weilin"

#通过writevip连到redis master修改value
10.18.14.94:6379> set test:username weilin123
OK
10.18.14.94:6379> get test:username
"weilin123"

#查看两台slave redis，已经复制同步成功
192.168.11.124:6379> get test:username
"weilin123"

192.168.11.125:6379> get test:username
"weilin123


#制造故障，shutdown redis master
[root@web01 redis]# redis-cli -h 192.168.11.123 -a 123456 shutdown
#过一会启动123 redis，原来的master变成slave
[root@web01 redis]# ./bin/redis-server ./conf/redis.conf


#sentinel failover 日志
[root@web01 ~]# tail -f /var/log/sentinel.log 
12504:X 16 May 2019 14:33:50.313 # Configuration loaded
12505:X 16 May 2019 14:33:50.319 * Increased maximum number of open files to 10032 (it was originally set to 1024).
12505:X 16 May 2019 14:33:50.320 * Running mode=sentinel, port=26379.
12505:X 16 May 2019 14:33:50.320 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
12505:X 16 May 2019 14:33:50.323 # Sentinel ID is 2e092b20d1e9c74ee0f56ab19ff53e88d74af528
12505:X 16 May 2019 14:33:50.323 # +monitor master master1 192.168.11.123 6379 quorum 2
12505:X 16 May 2019 14:33:50.325 * +slave slave 192.168.11.124:6379 192.168.11.124 6379 @ master1 192.168.11.123 6379
12505:X 16 May 2019 14:33:50.326 * +slave slave 192.168.11.125:6379 192.168.11.125 6379 @ master1 192.168.11.123 6379
12505:X 16 May 2019 14:33:56.737 * +sentinel sentinel 78a8068d31478ce7affab2ca754e7c31d600d3c2 192.168.11.124 26379 @ master1 192.168.11.123 6379
12505:X 16 May 2019 14:34:00.209 * +sentinel sentinel 67e6ccbf1395dfeeb96e1f36c022cd2abc170473 192.168.11.125 26379 @ master1 192.168.11.123 6379

12505:X 16 May 2019 14:58:48.244 # +sdown master master1 192.168.11.123 6379
12505:X 16 May 2019 14:58:48.279 # +new-epoch 1
12505:X 16 May 2019 14:58:48.281 # +vote-for-leader 67e6ccbf1395dfeeb96e1f36c022cd2abc170473 1
12505:X 16 May 2019 14:58:48.304 # +odown master master1 192.168.11.123 6379 #quorum 3/2
12505:X 16 May 2019 14:58:48.304 # Next failover delay: I will not start a failover before Thu May 16 14:59:48 2019
12505:X 16 May 2019 14:58:49.353 # +config-update-from sentinel 67e6ccbf1395dfeeb96e1f36c022cd2abc170473 192.168.11.125 26379 @ master1 192.168.11.123 6379
12505:X 16 May 2019 14:58:49.353 # +switch-master master1 192.168.11.123 6379 192.168.11.124 6379
12505:X 16 May 2019 14:58:49.354 * +slave slave 192.168.11.125:6379 192.168.11.125 6379 @ master1 192.168.11.124 6379
12505:X 16 May 2019 14:58:49.354 * +slave slave 192.168.11.123:6379 192.168.11.123 6379 @ master1 192.168.11.124 6379
12505:X 16 May 2019 14:58:54.395 # +sdown slave 192.168.11.123:6379 192.168.11.123 6379 @ master1 192.168.11.124 6379
12505:X 16 May 2019 15:00:33.211 # -sdown slave 192.168.11.123:6379 192.168.11.123 6379 @ master1 192.168.11.124 6379


[root@web02 redis]# tail -f /var/log/sentinel.log 
11919:X 16 May 2019 14:33:54.731 # Configuration loaded
11920:X 16 May 2019 14:33:54.737 * Increased maximum number of open files to 10032 (it was originally set to 1024).
11920:X 16 May 2019 14:33:54.739 * Running mode=sentinel, port=26379.
11920:X 16 May 2019 14:33:54.739 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
11920:X 16 May 2019 14:33:54.742 # Sentinel ID is 78a8068d31478ce7affab2ca754e7c31d600d3c2
11920:X 16 May 2019 14:33:54.742 # +monitor master master1 192.168.11.123 6379 quorum 2
11920:X 16 May 2019 14:33:54.744 * +slave slave 192.168.11.124:6379 192.168.11.124 6379 @ master1 192.168.11.123 6379
11920:X 16 May 2019 14:33:54.746 * +slave slave 192.168.11.125:6379 192.168.11.125 6379 @ master1 192.168.11.123 6379
11920:X 16 May 2019 14:33:56.508 * +sentinel sentinel 2e092b20d1e9c74ee0f56ab19ff53e88d74af528 192.168.11.123 26379 @ master1 192.168.11.123 6379
11920:X 16 May 2019 14:34:00.214 * +sentinel sentinel 67e6ccbf1395dfeeb96e1f36c022cd2abc170473 192.168.11.125 26379 @ master1 192.168.11.123 6379

11920:X 16 May 2019 14:58:48.205 # +sdown master master1 192.168.11.123 6379
11920:X 16 May 2019 14:58:48.284 # +new-epoch 1
11920:X 16 May 2019 14:58:48.286 # +vote-for-leader 67e6ccbf1395dfeeb96e1f36c022cd2abc170473 1
11920:X 16 May 2019 14:58:49.350 # +odown master master1 192.168.11.123 6379 #quorum 3/2
11920:X 16 May 2019 14:58:49.350 # Next failover delay: I will not start a failover before Thu May 16 14:59:48 2019
11920:X 16 May 2019 14:58:49.358 # +config-update-from sentinel 67e6ccbf1395dfeeb96e1f36c022cd2abc170473 192.168.11.125 26379 @ master1 192.168.11.123 6379
11920:X 16 May 2019 14:58:49.358 # +switch-master master1 192.168.11.123 6379 192.168.11.124 6379
11920:X 16 May 2019 14:58:49.358 * +slave slave 192.168.11.125:6379 192.168.11.125 6379 @ master1 192.168.11.124 6379
11920:X 16 May 2019 14:58:49.358 * +slave slave 192.168.11.123:6379 192.168.11.123 6379 @ master1 192.168.11.124 6379
11920:X 16 May 2019 14:58:54.455 # +sdown slave 192.168.11.123:6379 192.168.11.123 6379 @ master1 192.168.11.124 6379
11920:X 16 May 2019 15:00:33.194 # -sdown slave 192.168.11.123:6379 192.168.11.123 6379 @ master1 192.168.11.124 6379

12563:X 16 May 2019 14:33:58.203 # Configuration loaded
12564:X 16 May 2019 14:33:58.208 * Increased maximum number of open files to 10032 (it was originally set to 1024).
12564:X 16 May 2019 14:33:58.210 * Running mode=sentinel, port=26379.
12564:X 16 May 2019 14:33:58.210 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
12564:X 16 May 2019 14:33:58.212 # Sentinel ID is 67e6ccbf1395dfeeb96e1f36c022cd2abc170473
12564:X 16 May 2019 14:33:58.213 # +monitor master master1 192.168.11.123 6379 quorum 2
12564:X 16 May 2019 14:33:58.215 * +slave slave 192.168.11.124:6379 192.168.11.124 6379 @ master1 192.168.11.123 6379
12564:X 16 May 2019 14:33:58.217 * +slave slave 192.168.11.125:6379 192.168.11.125 6379 @ master1 192.168.11.123 6379
12564:X 16 May 2019 14:33:58.515 * +sentinel sentinel 2e092b20d1e9c74ee0f56ab19ff53e88d74af528 192.168.11.123 26379 @ master1 192.168.11.123 6379
12564:X 16 May 2019 14:33:58.795 * +sentinel sentinel 78a8068d31478ce7affab2ca754e7c31d600d3c2 192.168.11.124 26379 @ master1 192.168.11.123 6379




12564:X 16 May 2019 14:58:48.207 # +sdown master master1 192.168.11.123 6379
12564:X 16 May 2019 14:58:48.279 # +odown master master1 192.168.11.123 6379 #quorum 2/2
12564:X 16 May 2019 14:58:48.279 # +new-epoch 1
12564:X 16 May 2019 14:58:48.279 # +try-failover master master1 192.168.11.123 6379
12564:X 16 May 2019 14:58:48.282 # +vote-for-leader 67e6ccbf1395dfeeb96e1f36c022cd2abc170473 1
12564:X 16 May 2019 14:58:48.287 # 2e092b20d1e9c74ee0f56ab19ff53e88d74af528 voted for 67e6ccbf1395dfeeb96e1f36c022cd2abc170473 1
12564:X 16 May 2019 14:58:48.287 # 78a8068d31478ce7affab2ca754e7c31d600d3c2 voted for 67e6ccbf1395dfeeb96e1f36c022cd2abc170473 1
12564:X 16 May 2019 14:58:48.373 # +elected-leader master master1 192.168.11.123 6379
12564:X 16 May 2019 14:58:48.373 # +failover-state-select-slave master master1 192.168.11.123 6379
12564:X 16 May 2019 14:58:48.445 # +selected-slave slave 192.168.11.124:6379 192.168.11.124 6379 @ master1 192.168.11.123 6379
12564:X 16 May 2019 14:58:48.445 * +failover-state-send-slaveof-noone slave 192.168.11.124:6379 192.168.11.124 6379 @ master1 192.168.11.123 6379
12564:X 16 May 2019 14:58:48.535 * +failover-state-wait-promotion slave 192.168.11.124:6379 192.168.11.124 6379 @ master1 192.168.11.123 6379
12564:X 16 May 2019 14:58:49.293 # +promoted-slave slave 192.168.11.124:6379 192.168.11.124 6379 @ master1 192.168.11.123 6379
12564:X 16 May 2019 14:58:49.293 # +failover-state-reconf-slaves master master1 192.168.11.123 6379
12564:X 16 May 2019 14:58:49.355 * +slave-reconf-sent slave 192.168.11.125:6379 192.168.11.125 6379 @ master1 192.168.11.123 6379
12564:X 16 May 2019 14:58:50.375 * +slave-reconf-inprog slave 192.168.11.125:6379 192.168.11.125 6379 @ master1 192.168.11.123 6379
12564:X 16 May 2019 14:58:50.376 * +slave-reconf-done slave 192.168.11.125:6379 192.168.11.125 6379 @ master1 192.168.11.123 6379
12564:X 16 May 2019 14:58:50.475 # -odown master master1 192.168.11.123 6379
12564:X 16 May 2019 14:58:50.475 # +failover-end master master1 192.168.11.123 6379
12564:X 16 May 2019 14:58:50.475 # +switch-master master1 192.168.11.123 6379 192.168.11.124 6379
12564:X 16 May 2019 14:58:50.476 * +slave slave 192.168.11.125:6379 192.168.11.125 6379 @ master1 192.168.11.124 6379
12564:X 16 May 2019 14:58:50.476 * +slave slave 192.168.11.123:6379 192.168.11.123 6379 @ master1 192.168.11.124 6379
12564:X 16 May 2019 14:58:55.527 # +sdown slave 192.168.11.123:6379 192.168.11.123 6379 @ master1 192.168.11.124 6379
12564:X 16 May 2019 15:00:33.140 # -sdown slave 192.168.11.123:6379 192.168.11.123 6379 @ master1 192.168.11.124 6379
12564:X 16 May 2019 15:00:43.084 * +convert-to-slave slave 192.168.11.123:6379 192.168.11.123 6379 @ master1 192.168.11.124 6379

#从sentinel日志可以看到sentinel3获得failover领导权， web02上的slave提升为master,web01的原master重新启动后变成slave
#vip进行了漂移，web01的vip移除了，漂移到web02上
[root@web01 redis]# ip a
3: ens34: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:57:02:ab brd ff:ff:ff:ff:ff:ff
    inet 10.18.14.72/20 brd 10.18.15.255 scope global dynamic ens34
       valid_lft 36292sec preferred_lft 36292sec
    inet6 fe80::20c:29ff:fe57:2ab/64 scope link 
       valid_lft forever preferred_lft foreve

[root@web02 ~]# ip a
3: ens34: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:c1:47:f0 brd ff:ff:ff:ff:ff:ff
    inet 10.18.13.166/20 brd 10.18.15.255 scope global dynamic ens34
       valid_lft 31866sec preferred_lft 31866sec
    inet 10.18.14.94/20 scope global secondary ens34
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fec1:47f0/64 scope link 
       valid_lft forever preferred_lft forever

#连到vip上的客户端不影响读写
10.18.14.94:6379> get test:username
"weilin123"
10.18.14.94:6379> set test:username "weilin456"
OK

192.168.11.123:6379> get test:username
"weilin456"

192.168.11.124:6379> get test:username
"weilin456"

192.168.11.125:6379> get test:username
"weilin456"
```



## 部署负载均衡

为了不让负载均衡不成为瓶颈，采用LVS+DR的方式。

### redis节点

1. 在lo上添加vip

2. 加arp抑制的配置

   

#### web01

```
[root@web01 redis]# cd /etc/sysconfig/network-scripts
[root@web01 network-scripts]# vi ifcfg-lo:0
DEVICE=lo:0
IPADDR=10.18.14.93
NETMASK=255.255.255.255
ONBOOT=yes

[root@web01 network-scripts]# ifup lo:0
[root@web01 network-scripts]# ifup lo:0
[root@web01 network-scripts]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet 10.18.14.93/32 brd 10.18.14.93 scope global lo:0
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever

[root@web01 network-scripts]# echo "1" >/proc/sys/net/ipv4/conf/lo/arp_ignore
[root@web01 network-scripts]# echo "1" >/proc/sys/net/ipv4/conf/all/arp_ignore
[root@web01 network-scripts]# echo "2" >/proc/sys/net/ipv4/conf/lo/arp_announce
[root@web01 network-scripts]#  echo "2" >/proc/sys/net/ipv4/conf/all/arp_announce
```



#### web02

```
[root@web02 ~]# cd /etc/sysconfig/network-scripts
[root@web02 network-scripts]# vi ifcfg-lo:0
[root@web01 network-scripts]# vi ifcfg-lo:0
DEVICE=lo:0
IPADDR=10.18.14.93
NETMASK=255.255.255.255
ONBOOT=yes

[root@web02 network-scripts]# ifup lo:0
[root@web02 network-scripts]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet 10.18.14.93/32 brd 10.18.14.93 scope global lo:0
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever

[root@web02 network-scripts]# echo "1" >/proc/sys/net/ipv4/conf/lo/arp_ignore
[root@web02 network-scripts]# echo "1" >/proc/sys/net/ipv4/conf/all/arp_ignore
[root@web02 network-scripts]# echo "2" >/proc/sys/net/ipv4/conf/lo/arp_announce
[root@web02 network-scripts]# echo "2" >/proc/sys/net/ipv4/conf/all/arp_announce
```



#### web03

```
[root@web03 log]# cd /etc/sysconfig/network-scripts
[root@web03 network-scripts]# vi ifcfg-lo:0
DEVICE=lo:0
IPADDR=10.18.14.93
NETMASK=255.255.255.255
ONBOOT=yes

[root@web03 network-scripts]# ifup lo:0
[root@web03 network-scripts]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet 10.18.14.93/32 brd 10.18.14.93 scope global lo:0
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever

[root@web03 network-scripts]# echo "1" >/proc/sys/net/ipv4/conf/lo/arp_ignore
[root@web03 network-scripts]# echo "1" >/proc/sys/net/ipv4/conf/all/arp_ignore
[root@web03 network-scripts]# echo "2" >/proc/sys/net/ipv4/conf/lo/arp_announce
[root@web03 network-scripts]# echo "2" >/proc/sys/net/ipv4/conf/all/arp_announce
```



### keepalived节点

1. 配置keepalived.conf

注：实验环境网段不支持多播，所以vrrp配置采用单播方式



#### lb01

```
[root@lb01 ~]# yum install -y keepalived
[root@lb01 ~]# systemctl enable keepalived
[root@lb01 ~]# yum install ipvsadm -y
[root@lb01 ~]# lsmod|grep ip_vs 
ip_vs                 147456  0 
nf_conntrack          110592  1 ip_vs
libcrc32c              16384  2 xfs,ip_vs
#加载ipvs模块
[root@lb01 ~]# modprobe -- ip_vs_rr
[root@lb01 ~]# modprobe -- ip_vs_wrr
[root@lb01 ~]# lsmod | grep ip_vs   
ip_vs_wrr              16384  0 
ip_vs_rr               16384  0 
ip_vs                 147456  4 ip_vs_rr,ip_vs_wrr
nf_conntrack          110592  1 ip_vs
libcrc32c              16384  2 xfs,ip_vs

[root@lb01 ~]# cd /etc/keepalived/
[root@lb01 keepalived]# mv keepalived.conf keepalived.conf.template
[root@lb01 keepalived]# vi keepalived.conf
vrrp_instance VI_1 {
    state MASTER
    interface ens34
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        10.18.14.93
    }
    unicast_src_ip 10.18.14.54
    unicast_peer {
       10.18.14.64
    }
}

virtual_server 10.18.14.93 6379 {
    delay_loop 6
    lb_algo rr
    lb_kind DR
    persistence_timeout 0
    protocol TCP

    real_server 10.18.14.72 6379 {
        weight 1
        TCP_CHECK {
            connect_port 6379
            connect_timeout 2
            retry 3
            delay_before_retry 3
            
        }
    }

    real_server 10.18.13.166 6379 {
        weight 1
        TCP_CHECK {
            connect_port 6379
            connect_timeout 2
            retry 3
            delay_before_retry 3
            
        }
    }

    real_server 10.18.14.80 6379 {
        weight 1
        TCP_CHECK {
            connect_port 6379
            connect_timeout 2
            nb_get_retry 3
            delay_before_retry 3        
        }
    }
}

[root@lb01 keepalived]# systemctl restart keepalived                        
[root@lb01 keepalived]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:22:32:13 brd ff:ff:ff:ff:ff:ff
    inet 192.168.11.121/24 brd 192.168.11.255 scope global ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fe22:3213/64 scope link 
       valid_lft forever preferred_lft forever
3: ens34: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:22:32:1d brd ff:ff:ff:ff:ff:ff
    inet 10.18.14.54/20 brd 10.18.15.255 scope global dynamic ens34
       valid_lft 31844sec preferred_lft 31844sec
    inet 10.18.14.93/32 scope global ens34
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fe22:321d/64 scope link 
       valid_lft forever preferred_lft forever

[root@lb01 keepalived]# ipvsadm
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  lb01:6379 rr
  -> 10.18.13.166:6379            Route   1      0          0         
  -> 10.18.14.72:6379             Route   1      0          0         
  -> 10.18.14.80:6379             Route   1      1          0 
```



#### lb02

```
[root@lb02 ~]# yum install -y keepalived
[root@lb02 ~]# systemctl enable keepalived
[root@lb02 ~]# yum install ipvsadm -y
[root@lb02 ~]# lsmod|grep ip_vs 
ip_vs                 147456  0 
nf_conntrack          110592  1 ip_vs
libcrc32c              16384  2 xfs,ip_vs
#加载ipvs模块
[root@lb02 ~]# modprobe -- ip_vs_rr
[root@lb02 ~]# modprobe -- ip_vs_wrr
[root@lb02 ~]# lsmod | grep ip_vs   
ip_vs_wrr              16384  0 
ip_vs_rr               16384  0 
ip_vs                 147456  4 ip_vs_rr,ip_vs_wrr
nf_conntrack          110592  1 ip_vs
libcrc32c              16384  2 xfs,ip_vs

[root@lb02 ~]# cd /etc/keepalived/
[root@lb02 keepalived]# mv keepalived.conf keepalived.conf.template
[root@lb02 keepalived]# vi keepalived.conf
vrrp_instance VI_1 {
    state BACKUP
    interface ens34
    virtual_router_id 51
    priority 90
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        10.18.14.93
    }
    unicast_src_ip 10.18.14.64
    unicast_peer {
       10.18.14.54
    }
}

virtual_server 10.18.14.93 6379 {
    delay_loop 6
    lb_algo rr
    lb_kind DR
    persistence_timeout 0
    protocol TCP

    real_server 10.18.14.72 6379 {
        weight 1
        TCP_CHECK {
            connect_port 6379
            connect_timeout 2
            retry 3
            delay_before_retry 3
            
        }
    }

    real_server 10.18.13.166 6379 {
        weight 1
        TCP_CHECK {
            connect_port 6379
            connect_timeout 2
            retry 3
            delay_before_retry 3
            
        }
    }

    real_server 10.18.14.80 6379 {
        weight 1
        TCP_CHECK {
            connect_port 6379
            connect_timeout 2
            nb_get_retry 3
            delay_before_retry 3        
        }
    }
}

[root@lb01 keepalived]# systemctl restart keepalived
[root@lb02 keepalived]# systemctl start keepalived
[root@lb02 keepalived]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:72:52:0c brd ff:ff:ff:ff:ff:ff
    inet 192.168.11.122/24 brd 192.168.11.255 scope global ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fe72:520c/64 scope link 
       valid_lft forever preferred_lft forever
3: ens34: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:72:52:16 brd ff:ff:ff:ff:ff:ff
    inet 10.18.14.64/20 brd 10.18.15.255 scope global dynamic ens34
       valid_lft 39113sec preferred_lft 39113sec
    inet6 fe80::20c:29ff:fe72:5216/64 scope link 
       valid_lft forever preferred_lft forever

[root@lb02 keepalived]# ipvsadm
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.18.14.93:6379 rr
  -> 10.18.13.166:6379            Route   1      0          0         
  -> 10.18.14.72:6379             Route   1      0          0         
  -> 10.18.14.80:6379             Route   1      0          0
```



### 测试

```
# client通过readvip连到redis server，负载均衡到三台
#如果client 交互连接，就会一直连在某一台上
[root@devhost codis]# redis-cli -h 10.18.14.93 -a 123456 
10.18.14.93:6379> get test:username
"weilin456"
10.18.14.93:6379> get test:username
"weilin456"
10.18.14.93:6379> get test:username
"weilin456"

[root@lb01 keepalived]# ipvsadm
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  lb01:6379 rr
  -> 10.18.13.166:6379            Route   1      0          0         
  -> 10.18.14.72:6379             Route   1      0          0         
  -> 10.18.14.80:6379             Route   1      1          0         
[root@lb01 keepalived]# ipvsadm
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  lb01:6379 rr
  -> 10.18.13.166:6379            Route   1      0          0         
  -> 10.18.14.72:6379             Route   1      0          0         
  -> 10.18.14.80:6379             Route   1      1          0    

#如果client命令式连接，每次都轮询到一台上
[root@devhost codis]# redis-cli -h 10.18.14.93 -a 123456 get test:username
"weilin456"
[root@devhost codis]# redis-cli -h 10.18.14.93 -a 123456 get test:username
"weilin456"
[root@devhost codis]# redis-cli -h 10.18.14.93 -a 123456 get test:username
"weilin456"

[root@lb01 keepalived]# ipvsadm
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  lb01:6379 rr
  -> 10.18.13.166:6379            Route   1      0          1         
  -> 10.18.14.72:6379             Route   1      0          1         
  -> 10.18.14.80:6379             Route   1      0          2  

#主备切换
把lb01上的keepalived关闭，vip转移到lb02上

[root@lb01 keepalived]# systemctl stop keepalived
[root@lb01 keepalived]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:22:32:13 brd ff:ff:ff:ff:ff:ff
    inet 192.168.11.121/24 brd 192.168.11.255 scope global ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fe22:3213/64 scope link 
       valid_lft forever preferred_lft forever
3: ens34: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:22:32:1d brd ff:ff:ff:ff:ff:ff
    inet 10.18.14.54/20 brd 10.18.15.255 scope global dynamic ens34
       valid_lft 30975sec preferred_lft 30975sec
    inet6 fe80::20c:29ff:fe22:321d/64 scope link 
       valid_lft forever preferred_lft forever

[root@lb01 keepalived]# ipvsadm
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn


[root@lb02 keepalived]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:72:52:0c brd ff:ff:ff:ff:ff:ff
    inet 192.168.11.122/24 brd 192.168.11.255 scope global ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fe72:520c/64 scope link 
       valid_lft forever preferred_lft forever
3: ens34: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:72:52:16 brd ff:ff:ff:ff:ff:ff
    inet 10.18.14.64/20 brd 10.18.15.255 scope global dynamic ens34
       valid_lft 38379sec preferred_lft 38379sec
    inet 10.18.14.93/32 scope global ens34
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fe72:5216/64 scope link 
       valid_lft forever preferred_lft forever
#虽然vip漂移到lb02,但是redis-cli比较傻，不会重连，所以lb02的ipvs还没哟连接
[root@lb02 keepalived]# ipvsadm
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  lb02:6379 rr
  -> 10.18.13.166:6379            Route   1      0          0         
  -> 10.18.14.72:6379             Route   1      0          0         
  -> 10.18.14.80:6379             Route   1      0          0  

#当在redis-cli中发个命令，就连上了
10.18.14.93:6379> get test:username
"weilin456"

[root@lb02 keepalived]# ipvsadm
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  lb02:6379 rr
  -> 10.18.13.166:6379            Route   1      0          0         
  -> 10.18.14.72:6379             Route   1      0          0         
  -> 10.18.14.80:6379             Route   1      1          0
```

