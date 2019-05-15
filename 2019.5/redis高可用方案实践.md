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
masterauth shk9Hjwe
requirepass shk9Hjwe
maxmemory 1g
maxmemory-policy volatile-lru
appendonly yes

[root@web01 conf]# cd ..
[root@web01 redis]# ./bin/redis-server  ./conf/redis.conf

[root@web01 redis]# ss -tunlp|grep 6379
tcp    LISTEN     0      128       *:6379                  *:*                   users:(("redis-server",pid=11231,fd=7))
tcp    LISTEN     0      128      :::6379                 :::*                   users:(("redis-server",pid=11231,fd=6))

[root@web01 redis]# ./bin/redis-cli -a shk9Hjwe info


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

[root@web02 redis]#  ./bin/redis-cli -a shk9Hjwe info 

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
[root@web03 redis]# ./bin/redis-cli -a shk9Hjwe info
```



#### 测试主从复制功能

```
[root@devhost log]# redis-cli -h 192.168.11.123 -p 6379 -a shk9Hjwe  
192.168.11.123:6379> set test:username weilin
OK
192.168.11.123:6379> get test:username
"weilin"
192.168.11.123:6379> quit

[root@devhost log]# redis-cli -h 192.168.11.124 -p 6379 -a shk9Hjwe  
192.168.11.124:6379> get test:username
"weilin"
192.168.11.124:6379> quit

[root@devhost log]# redis-cli -h 192.168.11.125 -p 6379 -a shk9Hjwe  
192.168.11.125:6379> get test:username
"weilin"
192.168.11.125:6379> quit

```



### Sentinel

#### web01(sentinel1)

```
[root@web01 redis]# cd /etc
[root@web01 etc]# mkdir redis
[root@web01 etc]# cd redis
[root@web01 redis]# vi reconfig_master.sh
#!/bin/bash
MASTER_IP=$6  #第六个参数是新主redis的ip地址
LOCAL_IP='10.18.14.72'  #这里记得要改，每台服务器写自己的本地ip即可
VIP='10.18.14.94'
NETMASK='20'
INTERFACE='ens34'#网卡接口设备名称
if [ ${MASTER_IP} = ${LOCAL_IP} ];then   
    /usr/sbin/ip  addr  add ${VIP}/${NETMASK}  dev ${INTERFACE}  #将VIP绑定到该服务器上
    /usr/sbin/arping -q -c 3 -A ${VIP} -I ${INTERFACE}
    exit 0
else
   /usr/sbin/ip  addr del  ${VIP}/${NETMASK}  dev ${INTERFACE}   #将VIP从该服务器上删除
   exit 0
fi
exit 1

[root@web01 redis]# chmod u+x /etc/redis/reconfig_master.sh
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
sentinel auth-pass master1 shk9Hjwe
sentinel down-after-milliseconds mymaster 5000
sentinel parallel-syncs master1 1
sentinel failover-timeout master1 30000
sentinel client-reconfig-script master1 /etc/redis/reconfig_master.sh

[root@web01 redis]# ./bin/redis-server ./conf/sentinel.conf  --sentinel
[root@web01 redis]# ./bin/redis-cli -p 26379  info
```



#### web02(sentinel2)

```
[root@web02 redis]# mkdir -p /etc/redis
[root@web02 redis]# scp 192.168.11.123:/etc/redis/reconfig_master.sh /etc/redis/
[root@web02 redis]# vi /etc/redis/reconfig_master.sh
LOCAL_IP='10.18.13.166'

[root@web02 redis]# chmod u+x /etc/redis/reconfig_master.sh

[root@web02 redis]# scp 192.168.11.123:/usr/redis/conf/sentinel.conf /usr/redis/conf/
[root@web02 redis]# ./bin/redis-server ./conf/sentinel.conf  --sentinel
[root@web02 redis]# ./bin/redis-cli -p 26379 info

```



#### web03(sentinel3)

```
[root@web03 redis]#  mkdir -p /etc/redis
[root@web03 redis]# scp 192.168.11.123:/etc/redis/reconfig_master.sh /etc/redis/
[root@web03 redis]# vi /etc/redis/reconfig_master.sh
LOCAL_IP='10.18.14.80'

[root@web02 redis]# chmod u+x /etc/redis/reconfig_master.sh

[root@web03 redis]# scp 192.168.11.123:/usr/redis/conf/sentinel.conf /usr/redis/conf/
[root@web03 redis]# ./bin/redis-server ./conf/sentinel.conf  --sentinel
[root@web03 redis]# ./bin/redis-cli -p 26379  info
```



#### 故障转移测试



```
#客户端通过writevip连到redis master
[root@devhost codis]# redis-cli -h 10.18.14.94 -a shk9Hjwe
10.18.14.94:6379> get test:username
"weilin"

#客户端直接连到salve redis上
[root@devhost log]# redis-cli -h 192.168.11.124  -a shk9Hjwe         
192.168.11.124:6379> get test:username
"weilin"

[root@devhost ~]# redis-cli -h 192.168.11.125  -a shk9Hjwe  
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
[root@web01 redis]# ps -ef|grep redis
root      11231      1  0 20:07 ?        00:00:14 ./bin/redis-server *:6379
root      11312      1  0 20:54 ?        00:00:07 ./bin/redis-server *:26379 [sentinel]
root      11381   6924  0 21:16 pts/0    00:00:00 grep --color=auto redis
[root@web01 redis]# kill -9 11231

#sentinel failover 日志
[root@web01 ~]# tail -f /var/log/sentinel.log 
11311:X 15 May 2019 20:54:51.101 # Configuration loaded
11312:X 15 May 2019 20:54:51.168 * Increased maximum number of open files to 10032 (it was originally set to 1024).
11312:X 15 May 2019 20:54:51.170 * Running mode=sentinel, port=26379.
11312:X 15 May 2019 20:54:51.170 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
11312:X 15 May 2019 20:54:51.172 # Sentinel ID is c084ec33a5b60aa56d2ebd63a6b7442e2e257f38
11312:X 15 May 2019 20:54:51.172 # +monitor master master1 192.168.11.123 6379 quorum 2
11312:X 15 May 2019 20:54:51.174 * +slave slave 192.168.11.124:6379 192.168.11.124 6379 @ master1 192.168.11.123 6379
11312:X 15 May 2019 20:54:51.176 * +slave slave 192.168.11.125:6379 192.168.11.125 6379 @ master1 192.168.11.123 6379
11312:X 15 May 2019 20:58:48.606 * +sentinel sentinel 95e79e9566a8564c5336dd6f802851cde5b93984 192.168.11.124 26379 @ master1 192.168.11.123 6379
11312:X 15 May 2019 20:58:51.734 * +sentinel sentinel 6092c2d5a037ea0ba36a121e4e3c4ee8fda7cdb4 192.168.11.125 26379 @ master1 192.168.11.123 6379

11312:X 15 May 2019 21:16:28.260 # +sdown master master1 192.168.11.123 6379
11312:X 15 May 2019 21:16:28.313 # +odown master master1 192.168.11.123 6379 #quorum 3/2
11312:X 15 May 2019 21:16:28.313 # +new-epoch 1
11312:X 15 May 2019 21:16:28.313 # +try-failover master master1 192.168.11.123 6379
11312:X 15 May 2019 21:16:28.316 # +vote-for-leader c084ec33a5b60aa56d2ebd63a6b7442e2e257f38 1
11312:X 15 May 2019 21:16:28.322 # 6092c2d5a037ea0ba36a121e4e3c4ee8fda7cdb4 voted for c084ec33a5b60aa56d2ebd63a6b7442e2e257f38 1
11312:X 15 May 2019 21:16:28.323 # 95e79e9566a8564c5336dd6f802851cde5b93984 voted for c084ec33a5b60aa56d2ebd63a6b7442e2e257f38 1
11312:X 15 May 2019 21:16:28.380 # +elected-leader master master1 192.168.11.123 6379
11312:X 15 May 2019 21:16:28.380 # +failover-state-select-slave master master1 192.168.11.123 6379
11312:X 15 May 2019 21:16:28.457 # +selected-slave slave 192.168.11.125:6379 192.168.11.125 6379 @ master1 192.168.11.123 6379
11312:X 15 May 2019 21:16:28.457 * +failover-state-send-slaveof-noone slave 192.168.11.125:6379 192.168.11.125 6379 @ master1 192.168.11.123 6379
11312:X 15 May 2019 21:16:28.516 * +failover-state-wait-promotion slave 192.168.11.125:6379 192.168.11.125 6379 @ master1 192.168.11.123 6379
11312:X 15 May 2019 21:16:29.322 # +promoted-slave slave 192.168.11.125:6379 192.168.11.125 6379 @ master1 192.168.11.123 6379
11312:X 15 May 2019 21:16:29.323 # +failover-state-reconf-slaves master master1 192.168.11.123 6379
11312:X 15 May 2019 21:16:29.386 * +slave-reconf-sent slave 192.168.11.124:6379 192.168.11.124 6379 @ master1 192.168.11.123 6379
11312:X 15 May 2019 21:16:30.325 * +slave-reconf-inprog slave 192.168.11.124:6379 192.168.11.124 6379 @ master1 192.168.11.123 6379
11312:X 15 May 2019 21:16:30.325 * +slave-reconf-done slave 192.168.11.124:6379 192.168.11.124 6379 @ master1 192.168.11.123 6379
11312:X 15 May 2019 21:16:30.380 # +failover-end master master1 192.168.11.123 6379
11312:X 15 May 2019 21:16:30.380 # +switch-master master1 192.168.11.123 6379 192.168.11.125 6379
11312:X 15 May 2019 21:16:30.381 * +slave slave 192.168.11.124:6379 192.168.11.124 6379 @ master1 192.168.11.125 6379
11312:X 15 May 2019 21:16:30.381 * +slave slave 192.168.11.123:6379 192.168.11.123 6379 @ master1 192.168.11.125 6379
11312:X 15 May 2019 21:16:35.448 # +sdown slave 192.168.11.123:6379 192.168.11.123 6379 @ master1 192.168.11.125 6379
11312:X 15 May 2019 21:50:34.176 # -sdown slave 192.168.11.123:6379 192.168.11.123 6379 @ master1 192.168.11.125 6379


[root@web02 redis]# tail -f /var/log/sentinel.log 
11250:X 15 May 2019 20:58:46.548 # Configuration loaded
11251:X 15 May 2019 20:58:46.555 * Increased maximum number of open files to 10032 (it was originally set to 1024).
11251:X 15 May 2019 20:58:46.557 * Running mode=sentinel, port=26379.
11251:X 15 May 2019 20:58:46.557 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
11251:X 15 May 2019 20:58:46.559 # Sentinel ID is 95e79e9566a8564c5336dd6f802851cde5b93984
11251:X 15 May 2019 20:58:46.559 # +monitor master master1 192.168.11.123 6379 quorum 2
11251:X 15 May 2019 20:58:46.562 * +slave slave 192.168.11.124:6379 192.168.11.124 6379 @ master1 192.168.11.123 6379
11251:X 15 May 2019 20:58:46.564 * +slave slave 192.168.11.125:6379 192.168.11.125 6379 @ master1 192.168.11.123 6379
11251:X 15 May 2019 20:58:47.813 * +sentinel sentinel c084ec33a5b60aa56d2ebd63a6b7442e2e257f38 192.168.11.123 26379 @ master1 192.168.11.123 6379
11251:X 15 May 2019 20:58:51.737 * +sentinel sentinel 6092c2d5a037ea0ba36a121e4e3c4ee8fda7cdb4 192.168.11.125 26379 @ master1 192.168.11.123 6379

11251:X 15 May 2019 21:16:28.228 # +sdown master master1 192.168.11.123 6379
11251:X 15 May 2019 21:16:28.323 # +new-epoch 1
11251:X 15 May 2019 21:16:28.326 # +vote-for-leader c084ec33a5b60aa56d2ebd63a6b7442e2e257f38 1
11251:X 15 May 2019 21:16:29.383 # +odown master master1 192.168.11.123 6379 #quorum 3/2
11251:X 15 May 2019 21:16:29.384 # Next failover delay: I will not start a failover before Wed May 15 21:17:29 2019
11251:X 15 May 2019 21:16:29.396 # +config-update-from sentinel c084ec33a5b60aa56d2ebd63a6b7442e2e257f38 192.168.11.123 26379 @ master1 192.168.11.123 6379
11251:X 15 May 2019 21:16:29.396 # +switch-master master1 192.168.11.123 6379 192.168.11.125 6379
11251:X 15 May 2019 21:16:29.396 * +slave slave 192.168.11.124:6379 192.168.11.124 6379 @ master1 192.168.11.125 6379
11251:X 15 May 2019 21:16:29.397 * +slave slave 192.168.11.123:6379 192.168.11.123 6379 @ master1 192.168.11.125 6379
11251:X 15 May 2019 21:16:34.478 # +sdown slave 192.168.11.123:6379 192.168.11.123 6379 @ master1 192.168.11.125 6379
11251:X 15 May 2019 21:50:34.161 # -sdown slave 192.168.11.123:6379 192.168.11.123 6379 @ master1 192.168.11.125 6379
11251:X 15 May 2019 21:50:44.076 * +convert-to-slave slave 192.168.11.123:6379 192.168.11.123 6379 @ master1 192.168.11.125 6379

[root@web03 redis]# tail -f /var/log/sentinel.log 
11842:X 15 May 2019 20:58:49.668 # Configuration loaded
11843:X 15 May 2019 20:58:49.674 * Increased maximum number of open files to 10032 (it was originally set to 1024).
11843:X 15 May 2019 20:58:49.675 * Running mode=sentinel, port=26379.
11843:X 15 May 2019 20:58:49.675 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
11843:X 15 May 2019 20:58:49.678 # Sentinel ID is 6092c2d5a037ea0ba36a121e4e3c4ee8fda7cdb4
11843:X 15 May 2019 20:58:49.678 # +monitor master master1 192.168.11.123 6379 quorum 2
11843:X 15 May 2019 20:58:49.681 * +slave slave 192.168.11.124:6379 192.168.11.124 6379 @ master1 192.168.11.123 6379
11843:X 15 May 2019 20:58:49.682 * +slave slave 192.168.11.125:6379 192.168.11.125 6379 @ master1 192.168.11.123 6379
11843:X 15 May 2019 20:58:49.880 * +sentinel sentinel c084ec33a5b60aa56d2ebd63a6b7442e2e257f38 192.168.11.123 26379 @ master1 192.168.11.123 6379
11843:X 15 May 2019 20:58:50.632 * +sentinel sentinel 95e79e9566a8564c5336dd6f802851cde5b93984 192.168.11.124 26379 @ master1 192.168.11.123 6379

11843:X 15 May 2019 21:16:28.234 # +sdown master master1 192.168.11.123 6379
11843:X 15 May 2019 21:16:28.322 # +new-epoch 1
11843:X 15 May 2019 21:16:28.324 # +vote-for-leader c084ec33a5b60aa56d2ebd63a6b7442e2e257f38 1
11843:X 15 May 2019 21:16:28.336 # +odown master master1 192.168.11.123 6379 #quorum 2/2
11843:X 15 May 2019 21:16:28.336 # Next failover delay: I will not start a failover before Wed May 15 21:17:28 2019
11843:X 15 May 2019 21:16:29.391 # +config-update-from sentinel c084ec33a5b60aa56d2ebd63a6b7442e2e257f38 192.168.11.123 26379 @ master1 192.168.11.123 6379
11843:X 15 May 2019 21:16:29.391 # +switch-master master1 192.168.11.123 6379 192.168.11.125 6379
11843:X 15 May 2019 21:16:29.392 * +slave slave 192.168.11.124:6379 192.168.11.124 6379 @ master1 192.168.11.125 6379
11843:X 15 May 2019 21:16:29.392 * +slave slave 192.168.11.123:6379 192.168.11.123 6379 @ master1 192.168.11.125 6379
11843:X 15 May 2019 21:16:34.431 # +sdown slave 192.168.11.123:6379 192.168.11.123 6379 @ master1 192.168.11.125 6379
11843:X 15 May 2019 21:50:34.641 # -sdown slave 192.168.11.123:6379 192.168.11.123 6379 @ master1 192.168.11.125 6379

#从sentinel日志可以看到sentinel1获得failover领导权， web03上的slave提升为master


但是脚本有点问题，没调用，vip没有漂移
```

