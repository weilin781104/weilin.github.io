# 1. 介绍

负载均衡是一种把IP流量分发到一组实际服务器的方法。当设计负载均衡拓扑时，保证负载均衡器的可用性和保证它之后实际服务器的可用性一样重要。

Keepalived提供了负载均衡和高可用性的框架。负载均衡依赖著名且广泛使用的内核模块IPVS，它提供了四层网络的负载均衡。Keepalived实现了一套健康检查程序来根据实际server的健康情况动态地维护负载均衡服务器池。它自己的高可用通过VRRP实现。VRRP是用于路由器故障转移的基础。另外，Keepalived实现了一套VRRP状态机的钩子来提供底层告诉的协议交互。每个Keepalived框架可以单独使用也可以一起使用来提供弹性基础设施。

简而言之，Keepalived提供两个主要功能：

- LVS的健康检查
- 实现VRRPv2协议栈来处理负载均衡器的故障转移

# 2. 软件设计

待补充



# 3. 负载均衡技术

## 网络地址转换（NAT）

当负载均衡器有连长网卡时使用NAT，一张网卡是外网地址，一张网卡是内网地址。这种技术下，负载均衡器接收来自公网用户的请求，使用NAT（网络地址转换）把请求传递给内网的真实服务器。当真实服务器应答用户请求时，这些应答会进行反向转换。

一个优点是真实服务器隐藏在负载均衡器后而受到保护。另一个优点是节省IP地址，因为内网可以使用内网地址段。

主要的缺点是负载均衡器会变成瓶颈。它不仅要接收外网用户的请求并把应答回给外网用户；而且需要把请求传递给真实服务器并接收来自真实服务器的应答。

## 隧道（Tunneling）

在隧道模式中，负载均衡器通过IP隧道发送请求给真实服务器。

主要优点是可扩展性，负载均衡器把进来的请求转给真实服务器，而真实服务器把响应直接发给客户端，不需要再通过负载均衡器代理返回。它为处于不同网络的真实服务器提供了一种方法。

主要缺点是你需要获得最终工作环境投入的成本，因为它依赖你的网络架构。

## 直接路由（Direct Routing）

在直接路由模式中，用户发起请求给负载均衡器上的VIP。负载均衡器使用它预定义的分发算法把请求传递给恰当的真实服务器。不像NAT模式，真实服务器直接响应公网用户，绕过了负载均衡器。

主要优点是可扩展性，负载均衡器不需要负责把真实服务器的响应传递给外网用户。

缺点是ARP限制。为了让真实服务器直接响应外网用户请求，每个真实服务器必须使用VIP作为源地址发送应答。所以，VIP和MAC的组合在负载均衡器和真实服务器上都需要有，这会导致真实服务器会直接受到用户请求绕过了负载均衡器。有办法来解决这个问题，但是增加了配置的复杂度和客观理性。

# 安装Keepalived

可以从发布者的仓库安装Keepalived，也可以从源代码编译安装。

虽然从仓库安装是最快的方式，但是仓库中的keepalived版本会比最新稳定版落后几个版本。

## 从仓库安装

### 在Redhat Enterprise Liunx安装

自Redhat 6.4起，在Base仓库中已包括Keepalived安装包。所以，运行以下的命令来安装keepalived安装包和所有需要的依赖包。

```
yum install keepalived
```

### 在Debian安装

运行以下的命令来安装keepalived安装包和所有需要的依赖包。

```
apt-get install keepalived
```



## 从源代码编译安装

为了安装最新稳定版，需要从源代码编译。编译keepalived需要编译器、OpenSSL和Netlink库。你可以可选地安装Net-SNMP，它需要SNMP支持。

### 在RHEL/CentOS上预装

```
yum install curl gcc openssl-devel libnl3-devel net-snmp-devel
```

### 在Debian上预装

```
apt-get install curl gcc libssl-dev libnl-3-dev libnl-genl-3-dev libsnmp-dev
```

### 编译安装

使用curl或其它下载工具比如wget下载keepalived。软件在http://www.keepalived.org/download.html 或者 https://github.com/acassen/keepalived。 然后编译包：

```
curl --progress http://keepalived.org/software/keepalived-1.2.15.tar.gz | tar xz
cd keepalived-1.2.15
./configure
make
sudo make install
```

编译时一般推荐指定PREFIX，比如：

```
./configure --prefix=/usr/local/keepalived-1.2.15
```

这会方便keepalived 编译版本的卸载，只需要简单地删除安装目录即可。另外，这种安装方法允许多个keepalived版本共存而不会相互影响。使用symlink指向所需的版本。例如，你的目录结构可能看起来是这样的：

```
[root@lvs1 ~]# cd /usr/local
[root@lvs1 local]# ls -l
total 12
lrwxrwxrwx. 1 root root   17 Feb 24 20:23 keepalived -> keepalived-1.2.15
drwxr-xr-x. 2 root root 4096 Feb 24 20:22 keepalived-1.2.13
drwxr-xr-x. 2 root root 4096 Feb 24 20:22 keepalived-1.2.14
drwxr-xr-x. 2 root root 4096 Feb 24 20:22 keepalived-1.2.15
```

### 启动脚本

原官方文档使用的是init系统，我借用yum安装的systemd脚本。

```
vi /etc/systemd/system/keepalived.service
[Unit]
Description=LVS and VRRP High Availability Monitor
After=syslog.target network-online.target

[Service]
Type=forking
PIDFile=/var/run/keepalived.pid
KillMode=process
EnvironmentFile=-/etc/sysconfig/keepalived
ExecStart=/usr/sbin/keepalived $KEEPALIVED_OPTIONS
ExecReload=/bin/kill -HUP $MAINPID

[Install]
WantedBy=multi-user.target
```

注：PIDFile,EnvironmentFile，ExecStart需要根据实际情况修改。

# keepalived配置概要

## 全局定义概要

```
global_defs {
    notification_email {
        email
        email
    }
    notification_email_from email
    smtp_server host
    smtp_connect_timeout num
    lvs_id string
}
```

| 关键字                  | 定义                                       | 类型      |
| ----------------------- | ------------------------------------------ | --------- |
| global_defs             | 定义全局定义配置块                         |           |
| notification_email      | 接收通知邮件的邮箱地址                     | List      |
| notification_email_from | 当处理"MAIL FROM:"SMTP命令时使用的邮箱地址 | List      |
| smtp_server             | 发送通知邮件的邮箱服务器                   | alphanum  |
| smtp_connection_timeout | SMTP流处理的过期时间                       | numerical |
| lvs_id                  | 指定LVS导向器的名字                        | alphanum  |

Email 类型：符合SMTP规范字符串：比如“user@domain.com”

## 虚拟服务器定义概要

```
virtual_server (@IP PORT)|(fwmark num) {
    delay_loop num
    lb_algo rr|wrr|lc|wlc|sh|dh|lblc
    lb_kind NAT|DR|TUN
    (nat_mask @IP)
    persistence_timeout num
    persistence_granularity @IP
    virtualhost string
    protocol TCP|UDP

    sorry_server @IP PORT
    real_server @IP PORT {
        weight num
        TCP_CHECK {
            connect_port num
            connect_timeout num
        }
    }
    real_server @IP PORT {
        weight num
        MISC_CHECK {
            misc_path /path_to_script/script.sh
            (or misc_path “ /path_to_script/script.sh <arg_list>”)
        }
    }
}
real_server @IP PORT {
    weight num
    HTTP_GET|SSL_GET {
        url { # You can add multiple url block
            path alphanum
            digest alphanum
        }
        connect_port num
        connect_timeout num
        retry num
        delay_before_retry num
    }
}
```

| 关键字                  | 定义                                                       | 类型      |
| ----------------------- | ---------------------------------------------------------- | --------- |
| virtual_server          | 定义一个虚拟服务器定义块                                   |           |
| fwmark                  | 指定虚拟服务器是FWMARK                                     |           |
| delay_loop              | 指定健康检查的间隔（以秒为单位）                           | numerical |
| lb_algo                 | 指定特定的负载均衡算法（rr\|wrr\|lc\|wlc...）              | string    |
| lb_kind                 | 指定负载均衡转发模式（NAT\|DR\|TUN）                       | string    |
| persistence_timeout     | 指定连接持久的时间，即某个客户端指向特定真实服务器的锁定期 | numerical |
| persistence_granularity | 指定持久连接的粒度掩码（子网掩码）                         |           |
| virtualhost             | 指定用于HTTP\|SSL_GET的HTTP虚拟主机                        | alphanum  |
| protocol                | 指定协议种类（TCP\|UDP）                                   | numerical |
| sorry_server            | 如果所有真实服务器都宕，被加到调度池的服务器               |           |
| real_server             | 指定真实服务器成员                                         |           |
| weight                  | 指定真实服务器的权重，用于负载均衡决策                     | numerical |
| TCP_CHECK               | 使用TCP连接检查真实服务器可用性                            |           |
| MISC_CHECK              | 使用用户定义脚本检查真实服务器可用性                       |           |
| misc_path               | 指定执行的脚本（全路径）                                   | path      |
| HTTP_GET                | 使用HTTP GET请求检查真实服务器可用性                       |           |
| SSL_GET                 | 使用SSL GET请求检查真实服务器可用性                        |           |
| url                     | 指定url定义块                                              |           |
| path                    | 指定url路径                                                | alphanum  |
| digest                  | 指定特定url路径的摘要                                      | alphanum  |
| connect_port            | 连接远端服务器的特定TCP端口                                | numerical |
| connect_timeout         | 连接远端服务器的超时                                       | numerical |
| retry                   | 重试最大次数                                               | numerical |
| delay_before_retry      | 两次连续重试间的延迟间隔                                   | numerical |

注：

如果你不是使用linux 内核2.2系列的LVS，关键字nat_mask过时了。这个标志可以用来定义反向NAT的粒度。

注：

当前，健康检查只实现了TCP协议用于服务监控。

注：

类型path指脚本调用的全路径。Note that for scripts requiring arguments the path and arguments must be enclosed in double quotes (”).



## VRRP实例定义概要

```
vrrp_sync_group string {
    group {
        string
        string
    }
    notify_master /path_to_script/script_master.sh
        (or notify_master “ /path_to_script/script_master.sh <arg_list>”)
    notify_backup /path_to_script/script_backup.sh
        (or notify_backup “/path_to_script/script_backup.sh <arg_list>”)
    notify_fault /path_to_script/script_fault.sh
        (or notify_fault “ /path_to_script/script_fault.sh <arg_list>”)
}
vrrp_instance string {
    state MASTER|BACKUP
    interface string
    mcast_src_ip @IP
    lvs_sync_daemon_interface string
    virtual_router_id num
    priority num
    advert_int num
    smtp_alert
    authentication {
        auth_type PASS|AH
        auth_pass string
    }
    virtual_ipaddress { # Block limited to 20 IP addresses
        @IP
        @IP
        @IP
    }
    virtual_ipaddress_excluded { # Unlimited IP addresses
        @IP
        @IP
        @IP
    }
    notify_master /path_to_script/script_master.sh
        (or notify_master “ /path_to_script/script_master.sh <arg_list>”)
    notify_backup /path_to_script/script_backup.sh
        (or notify_backup “ /path_to_script/script_backup.sh <arg_list>”)
    notify_fault /path_to_script/script_fault.sh
        (or notify_fault “ /path_to_script/script_fault.sh <arg_list>”)
}
```



| 关键字                    | 定义                                  | 类型   |
| ------------------------- | ------------------------------------- | ------ |
| vrrp_instance             | 定义一个VRRP实例定义块                |        |
| state                     | 指定实例状态                          |        |
| interface                 | 指定实例运行的网卡                    | string |
| mcast_src_ip              | 指定VRRP广播IP头部的源IP地址          |        |
| lvs_sync_daemon_interface | 指定LVS sync_daemon运行的网卡         | string |
| virtual_router_id         | 指定实例所属的VRRP router id          | number |
| priority                  | 指定实例的优先级                      | number |
| advert_int                | 指定广播间隔（以秒为单位）            | number |
| smtp_alert                | 激活MASTER状态切换的SMTP通知          |        |
| authentication            | 指定VRRP认证定义块                    |        |
| auth_type                 | 指定使用的认证种类（PASS\|AH）        |        |
| auth_pass                 | 指定使用的密码                        | string |
| virtual_ipaddress         | 指定VRRP VIP定义块                    |        |
| vitual_ipaddress_excluded | 指定排除的VRRP VIP定义块              |        |
| notify_master             | 指定切换到master状态时执行的shell脚本 | path   |
| notify_backup             | 指定切换到backup状态时执行的shell脚本 | path   |
| notify_fault              | 指定切换到fault状态时执行的shell脚本  | path   |
| vrrp_sync_group           | 指定VRRP同步实例组                    | string |

path类型：脚本的系统路径，比如"/usr/local/bin/transit.sh <arg_list>"

# Keepalived程序概要

keepalived程序包有两个程序。

## Keepalived daemon

keepalived命令行参数：

-f, –use-file=FILE

使用指定的配置文件。默认配置文件是 “/etc/keepalived/keepalived.conf”.

-P, –vrrp

只运行VRRP子系统。这对不使用IPVS负载均衡器的配置有用。

-C, –check

只运行健康检查子系统。这对使用IPVS负载均衡器，并且只分发不故障转移有用

-l, –log-console

记录日志到本地控制台。默认是记录日志到syslog。

-D, –log-detail

记录详细日志

-S, –log-facility=[0-7]

设置syslog 为LOG_LOCAL[0-7]. 默认为 LOG_DAEMON.

-V, –dont-release-vrrp

Don’t remove VRRP VIPs and VROUTEs on daemon stop. The default behavior is to remove all VIPs and VROUTEs when keepalived exits

-I, –dont-release-ipvs

Don’t remove IPVS topology on daemon stop. The default behavior it to remove all entries from the IPVS virtual server table on when keepalived exits.

-R, –dont-respawn

Don’t respawn child processes. The default behavior is to restart the VRRP and checker processes if either process exits.

-n, –dont-fork

不派生daemon进程。该选项使得keepalived运行在前台。

-d, –dump-conf

保存内存中的配置数据到日志文件

-p, –pid=FILE

为keepalived父进程指定pidfile。默认的pidfile 是“/var/run/keepalived.pid”.

-r, –vrrp_pid=FILE

为VRRP子进程指定pidfile。默认的pidfile是 “/var/run/keepalived_vrrp.pid”.

-c, –checkers_pid=FILE

为健康检查子进程指定pidfile。默认的pidfile是“/var/run/keepalived_checkers.pid”

-x, –snmp

启用SNMP子系统

-v, –version

显示版本并退出

-h, –help

显示帮助信息并退出



## genhash工具

The `genhash` binary is used to generate digest strings. The genhash command line arguments are:

- –use-ssl, -S

  Use SSL to connect to the server.

- –server <host>, -s

  Specify the ip address to connect to.

- –port <port>, -p

  Specify the port to connect to.

- –url <url>, -u

  Specify the path to the file you want to generate the hash of.

- –use-virtualhost <host>, -V

  Specify the virtual host to send along with the HTTP headers.

- –hash <alg>, -H

  Specify the hash algorithm to make a digest of the target page. Consult the help screen for list of available ones with a mark of the default one.

- –verbose, -v

  Be verbose with the output.

- –help, -h

  Display the program help screen and exit.

- –release, -r

  Display the release number (version) and exit.

## Running Keepalived daemon

To run Keepalived simply type:

```
[root@lvs tmp]# /etc/rc.d/init.d/keepalived.init start
Starting Keepalived for LVS:                            [ OK ]
```

注：这是init。如果是systemd的话 systemctl start keepalived。



所有日志记录在linux syslog。如果你启动keepalived使用“dump configuration data” 选项，你会看到/var/log/messages (在Debian是*/var/log/daemon.log* 根据syslog配置) 如下:

```
Jun 7 18:17:03 lvs1 Keepalived: Starting Keepalived v0.6.1 (06/13, 2002)
Jun 7 18:17:03 lvs1 Keepalived: Configuration is using : 92013 Bytes
Jun 7 18:17:03 lvs1 Keepalived: ------< Global definitions >------
Jun 7 18:17:03 lvs1 Keepalived: LVS ID = LVS_PROD
Jun 7 18:17:03 lvs1 Keepalived: Smtp server = 192.168.200.1
Jun 7 18:17:03 lvs1 Keepalived: Smtp server connection timeout = 30
Jun 7 18:17:03 lvs1 Keepalived: Email notification from = keepalived@domain.com
Jun 7 18:17:03 lvs1 Keepalived: Email notification = alert@domain.com
Jun 7 18:17:03 lvs1 Keepalived: Email notification = 0633556699@domain.com
Jun 7 18:17:03 lvs1 Keepalived: ------< SSL definitions >------
Jun 7 18:17:03 lvs1 Keepalived: Using autogen SSL context
Jun 7 18:17:03 lvs1 Keepalived: ------< LVS Topology >------
Jun 7 18:17:03 lvs1 Keepalived: System is compiled with LVS v0.9.8
Jun 7 18:17:03 lvs1 Keepalived: VIP = 10.10.10.2, VPORT = 80
Jun 7 18:17:03 lvs1 Keepalived: VirtualHost = www.domain1.com
Jun 7 18:17:03 lvs1 Keepalived: delay_loop = 6, lb_algo = rr
Jun 7 18:17:03 lvs1 Keepalived: persistence timeout = 50
Jun 7 18:17:04 lvs1 Keepalived: persistence granularity = 255.255.240.0
Jun 7 18:17:04 lvs1 Keepalived: protocol = TCP
Jun 7 18:17:04 lvs1 Keepalived: lb_kind = NAT
Jun 7 18:17:04 lvs1 Keepalived: sorry server = 192.168.200.200:80
Jun 7 18:17:04 lvs1 Keepalived: RIP = 192.168.200.2, RPORT = 80, WEIGHT = 1
Jun 7 18:17:04 lvs1 Keepalived: RIP = 192.168.200.3, RPORT = 80, WEIGHT = 2
Jun 7 18:17:04 lvs1 Keepalived: VIP = 10.10.10.3, VPORT = 443
Jun 7 18:17:04 lvs1 Keepalived: VirtualHost = www.domain2.com
Jun 7 18:17:04 lvs1 Keepalived: delay_loop = 3, lb_algo = rr
Jun 7 18:17:04 lvs1 Keepalived: persistence timeout = 50
Jun 7 18:17:04 lvs1 Keepalived: protocol = TCP
Jun 7 18:17:04 lvs1 Keepalived: lb_kind = NAT
Jun 7 18:17:04 lvs1 Keepalived: RIP = 192.168.200.4, RPORT = 443, WEIGHT = 1
Jun 7 18:17:04 lvs1 Keepalived: RIP = 192.168.200.5, RPORT = 1358, WEIGHT = 1
Jun 7 18:17:05 lvs1 Keepalived: ------< Health checkers >------
Jun 7 18:17:05 lvs1 Keepalived: 192.168.200.2:80
Jun 7 18:17:05 lvs1 Keepalived: Keepalive method = HTTP_GET
Jun 7 18:17:05 lvs1 Keepalived: Connection timeout = 3
Jun 7 18:17:05 lvs1 Keepalived: Nb get retry = 3
Jun 7 18:17:05 lvs1 Keepalived: Delay before retry = 3
Jun 7 18:17:05 lvs1 Keepalived: Checked url = /testurl/test.jsp,
Jun 7 18:17:05 lvs1 Keepalived: digest = 640205b7b0fc66c1ea91c463fac6334d
Jun 7 18:17:05 lvs1 Keepalived: 192.168.200.3:80
Jun 7 18:17:05 lvs1 Keepalived: Keepalive method = HTTP_GET
Jun 7 18:17:05 lvs1 Keepalived: Connection timeout = 3
Jun 7 18:17:05 lvs1 Keepalived: Nb get retry = 3
Jun 7 18:17:05 lvs1 Keepalived: Delay before retry = 3
Jun 7 18:17:05 lvs1 Keepalived: Checked url = /testurl/test.jsp,
Jun 7 18:17:05 lvs1 Keepalived: digest = 640205b7b0fc66c1ea91c463fac6334c
Jun 7 18:17:05 lvs1 Keepalived: Checked url = /testurl2/test.jsp,
Jun 7 18:17:05 lvs1 Keepalived: digest = 640205b7b0fc66c1ea91c463fac6334c
Jun 7 18:17:06 lvs1 Keepalived: 192.168.200.4:443
Jun 7 18:17:06 lvs1 Keepalived: Keepalive method = SSL_GET
Jun 7 18:17:06 lvs1 Keepalived: Connection timeout = 3
Jun 7 18:17:06 lvs1 Keepalived: Nb get retry = 3
Jun 7 18:17:06 lvs1 Keepalived: Delay before retry = 3
Jun 7 18:17:06 lvs1 Keepalived: Checked url = /testurl/test.jsp,
Jun 7 18:17:05 lvs1 Keepalived: digest = 640205b7b0fc66c1ea91c463fac6334d
Jun 7 18:17:06 lvs1 Keepalived: Checked url = /testurl2/test.jsp,
Jun 7 18:17:05 lvs1 Keepalived: digest = 640205b7b0fc66c1ea91c463fac6334d
Jun 7 18:17:06 lvs1 Keepalived: 192.168.200.5:1358
Jun 7 18:17:06 lvs1 Keepalived: Keepalive method = TCP_CHECK
Jun 7 18:17:06 lvs1 Keepalived: Connection timeout = 3
Jun 7 18:17:06 lvs1 Keepalived: Registering Kernel netlink reflector
```



# IPVS调度算法

The following scheduling algorithms are supported by the IPVS kernel code. (heavily stolen from LVS website).

下面是IPVS内核支持的调度算法（摘自LVS网站）

## 轮询 (rr)

轮询调度算法发送每个进来的请求给服务器列表中的下一个服务器。如果在三个服务器的集群（服务器A,B和C）中，请求1会去服务器A，请求2会去服务器B，请求3会去服务器C，请求4会去服务器A...，这样完成服务器的循环或轮询。无论进来的连接数或者每台服务器的响应时间，它公平地对待所有服务器。相对传统轮询DNS有一些优势。轮询DNS解析单域名到不同的IP地址，调度粒度是基于主机的，并且DNS缓存阻碍了基本算法，因此这些因素导致了在真实服务器间严重的动态负载不均衡。而IPVS的调度粒度是基于网络连接的，由于优良的调度粒度它比轮询DNS更优。

## 加权重的轮询 (wrr)

加权重的轮询调度能更好地应对不同处理能力的服务器。每个服务器设置一个权重--表示处理能力的整数。高权重的服务器比低权重的服务器会先收到新连接并会获得更多的连接，同权重的服务器会得到相同的连接。例如，真实服务器A,B和C分别权重为4,3,2，好的调度顺序会是AABABCABC（在一个调度周期内）。在加权重的轮询调度实现中，调度顺序会在IPVS规则修改后根据服务器权重生成。网络连接以轮询的方式基于调度顺序被定向到不同的真实服务器。

当真实服务器的处理能力不同时，加权重的轮询调度比轮询调度更好。但是当请求负载变化剧烈时，它会在真实服务器间引起动态负载不均衡。简而言之，大多数请求可能会被定向到相同的真实服务器。(In short, there is the possibility that a majority of requests requiring large responses may be directed to the same real server.)

实际上，轮询算法是种特殊的加权重的轮询算法，就是它的权重值都相同。



## Least Connection (lc)

The least-connection scheduling algorithm directs network connections to the server with the least number of established connections. This is one of the dynamic scheduling algorithms; because it needs to count live connections for each server dynamically. For a Virtual Server that is managing a collection of servers with similar performance, least-connection scheduling is good to smooth distribution when the load of requests vary a lot. Virtual Server will direct requests to the real server with the fewest active connections.

At a first glance it might seem that least-connection scheduling can also perform well even when there are servers of various processing capacities, because the faster server will get more network connections. In fact, it cannot perform very well because of the TCP’s TIME_WAIT state. The TCP’s TIME_WAIT is usually 2 minutes, during this 2 minutes a busy web site often receives thousands of connections, for example, the server A is twice as powerful as the server B, the server A is processing thousands of requests and keeping them in the TCP’s TIME_WAIT state, but server B is crawling to get its thousands of connections finished. So, the least-connection scheduling cannot get load well balanced among servers with various processing capacities.

## Weighted Least Connection (wlc)

The weighted least-connection scheduling is a superset of the least-connection scheduling, in which you can assign a performance weight to each real server. The servers with a higher weight value will receive a larger percentage of live connections at any one time. The Virtual Server Administrator can assign a weight to each real server, and network connections are scheduled to each server in which the percentage of the current number of live connections for each server is a ratio to its weight. The default weight is one.

The weighted least-connections scheduling works as follows:

> Supposing there is n real servers, each server i has weight Wi (i=1,..,n), and alive connections Ci (i=1,..,n), ALL_CONNECTIONS is the sum of Ci (i=1,..,n), the next network connection will be directed to the server j, in which
>
> > (Cj/ALL_CONNECTIONS)/Wj = min { (Ci/ALL_CONNECTIONS)/Wi } (i=1,..,n)
>
> Since the ALL_CONNECTIONS is a constant in this lookup, there is no need to divide Ci by ALL_CONNECTIONS, it can be optimized as
>
> > Cj/Wj = min { Ci/Wi } (i=1,..,n)

The weighted least-connection scheduling algorithm requires additional division than the least-connection. In a hope to minimize the overhead of scheduling when servers have the same processing capacity, both the least-connection scheduling and the weighted least-connection scheduling algorithms are implemented.

## Locality-Based Least Connection (lblc)

The locality-based least-connection scheduling algorithm is for destination IP load balancing. It is usually used in cache cluster. This algorithm usually directs packet destined for an IP address to its server if the server is alive and under load. If the server is overloaded (its active connection numbers is larger than its weight) and there is a server in its half load, then allocate the weighted least-connection server to this IP address.

## Locality-Based Least Connection with Replication (lblcr)

The locality-based least-connection with replication scheduling algorithm is also for destination IP load balancing. It is usually used in cache cluster. It differs from the LBLC scheduling as follows: the load balancer maintains mappings from a target to a set of server nodes that can serve the target. Requests for a target are assigned to the least-connection node in the target’s server set. If all the node in the server set are over loaded, it picks up a least-connection node in the cluster and adds it in the sever set for the target. If the server set has not been modified for the specified time, the most loaded node is removed from the server set, in order to avoid high degree of replication.

## Destination Hashing (dh)

The destination hashing scheduling algorithm assigns network connections to the servers through looking up a statically assigned hash table by their destination IP addresses.

## Source Hashing (sh)

The source hashing scheduling algorithm assigns network connections to the servers through looking up a statically assigned hash table by their source IP addresses.

## Shortest Expected Delay (seq)

The shortest expected delay scheduling algorithm assigns network connections to the server with the shortest expected delay. The expected delay that the job will experience is (Ci + 1) / Ui if sent to the ith server, in which Ci is the number of connections on the the ith server and Ui is the fixed service rate (weight) of the ith server.

## Never Queue (nq)

The never queue scheduling algorithm adopts a two-speed model. When there is an idle server available, the job will be sent to the idle server, instead of waiting for a fast one. When there is no idle server available, the job will be sent to the server that minimize its expected delay (The Shortest Expected Delay scheduling algorithm).

## Overflow-Connection (ovf)

The Overflow connection scheduling algorithm implements “overflow” loadbalancing according to number of active connections , will keep all conections to the node with the highest weight and overflow to the next node if the number of connections exceeds the node’s weight. Note that this scheduler might not be suitable for UDP because it only uses active connections





# 案例研究：健康检查



As an example we can introduce the following LVS topology:

First of all you need a well-configured LVS topology. In the rest of this document, we will assume that all system configurations have been done. This kind of topology is generally implemented in a DMZ architecture. For more information on LVS NAT topology and system configuration please read the nice Joseph Mack LVS HOWTO.

## Main architecture components

- LVS Router: Owning the load balanced IP Class routed (192.168.100.0/24).
- Network Router : The default router for the entire internal network. All the LAN workstations are handled through this IP address.
- Network DNS Server: Referencing the internal network IP topology.
- SMTP Server: SMTP server receiving the mail alerts.
- SERVER POOL: Set of servers hosting load balanced services.

## Server pool specifications

In this sample configuration we have 2 server pools:

- Server pool 1: Hosting the HTTP & SSL services. Each server owns two application servers (IBM WEBSPHERE & BEA WEBLOGIC)
- Server pool 2: Hosting the SMTP service.

## Keepalived configuration

You are now ready to configure the Keepalived daemon according to your LVS topology. The whole configuration is done in the /etc/keepalived/keepalived.conf file. In our case study this file looks like:

```
# Configuration File for keepalived
global_defs {
    notification_email {
        admin@domain.com
        0633225522@domain.com
    }
    notification_email_from keepalived@domain.com
    smtp_server 192.168.200.20
    smtp_connect_timeout 30
    lvs_id LVS_MAIN
}
virtual_server 192.168.200.15 80 {
    delay_loop 30
    lb_algo wrr
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

    sorry_server 192.168.100.100 80

    real_server 192.168.100.2 80 {
        weight 2
        HTTP_GET {
            url {
                path /testurl/test.jsp
                digest ec90a42b99ea9a2f5ecbe213ac9eba03
            }
            url {
                path /testurl2/test.jsp
                digest 640205b7b0fc66c1ea91c463fac6334c
            }
            connect_timeout 3
            retry 3
            delay_before_retry 2
        }
    }
    real_server 192.168.100.3 80 {
        weight 1
        HTTP_GET {
            url {
                path /testurl/test.jsp
                digest 640205b7b0fc66c1ea91c463fac6334c
            }
            connect_timeout 3
            retry 3
            delay_before_retry 2
        }
    }
}
virtual_server 192.168.200.15 443 {
    delay_loop 20
    lb_algo rr
    lb_kind NAT
    persistence_timeout 360
    protocol TCP
    real_server 192.168.100.2 443 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
        }
    }
    real_server 192.168.100.3 443 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
        }
    }
}
virtual_server 192.168.200.15 25 {
    delay_loop 15
    lb_algo wlc
    lb_kind NAT
    persistence_timeout 50
    protocol TCP
    real_server 192.168.100.4 25 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
        }
    }
    real_server 192.168.100.5 25 {
        weight 2
        TCP_CHECK {
            connect_timeout 3
        }
    }
}
```

According to this configuration example, the Keepalived daemon will drive the kernel using following information:

- The LVS server will own the name: LVS_MAIN

- Notification:

  > - SMTP server will be: 192.168.200.20
  > - SMTP connection timeout is set to: 30 seconded
  > - Notification emails will be: [admin@domain.com](mailto:admin%40domain.com) & [0633225522@domain.com](mailto:0633225522%40domain.com)

- Load balanced services:

  > - HTTP: VIP 192.168.200.15 port 80
  >
  >   > - Load balancing: Using Weighted Round Robin scheduler with NAT forwarding. Connection persistence is set to 50 seconds on each TCP service. If you are using Linux kernel 2.2 you need to specify the NAT netmask to define the IPFW masquerade granularity (nat_mask keyword). The delay loop is set to 30 seconds
  >   > - Sorry Server: If all real servers are removed from the VS’s server pools, we add the sorry_server 192.168.100.100 port 80 to serve clients requests.
  >   > - Real server 192.168.100.2 port 80 will be weighted to 2. Failure detection will be based on HTTP_GET over 2 URLS. The service connection timeout will be set to 3 seconds. The real server will be considered down after 3 retries. The daemon will wait for 2 seconds before retrying.
  >   > - Real server 192.168.100.3 port 80 will be weighted to 1. Failure detection will be based on HTTP_GET over 1 URL. The service connection timeout will be set to 3 seconds. The real server will be considered down after 3 retries. The daemon will wait for 2 seconds before retrying.
  >
  > - SSL: VIP 192.168.200.15 port 443
  >
  >   > - Load balancing: Using Round Robin scheduler with NAT forwarding. Connection persistence is set to 360 seconds on each TCP service. The delay loop is set to 20 seconds
  >   > - Real server 192.168.100.2 port 443 will be weighted to 2. Failure detection will be based on TCP_CHECK. The real server will be considered down after a 3 second connection timeout.
  >   > - Real server 192.168.100.3 port 443 will be weighted to 2. Failure detection will be based on TCP_CHECK. The real server will be considered down after a 3 second connection timeout.
  >
  > - SMTP: VIP 192.168.200.15 port 25
  >
  >   > - Load balancing: Using Weighted Least Connection scheduling algorithm in a NAT topology with connection persistence set to 50 seconds. The delay loop is set to 15 seconds
  >   > - Real server 192.168.100.4 port 25 will be weighted to 1. Failure detection will be based on TCP_CHECK. The real server will be considered down after a 3 second connection timeout.
  >   > - Real server 192.168.100.5 port 25 will be weighted to 2. Failure detection will be based on TCP_CHECK. The real server will be considered down after a 3 second connection timeout.

For SSL server health check, we can use SSL_GET checkers. The configuration block for a corresponding real server will look like:

```
virtual_server 192.168.200.15 443 {
    delay_loop 20
    lb_algo rr
    lb_kind NAT
    persistence_timeout 360
    protocol TCP
    real_server 192.168.100.2 443 {
        weight 1
        SSL_GET
        {
            url {
                path /testurl/test.jsp
                digest ec90a42b99ea9a2f5ecbe213ac9eba03
            }
            url {
                path /testurl2/test.jsp
                digest 640205b7b0fc66c1ea91c463fac6334c
            }
            connect_timeout 3
            retry 3
            delay_before_retry 2
        }
    }
    real_server 192.168.100.3 443 {
        weight 1
        SSL_GET
        {
            url {
                path /testurl/test.jsp
                digest 640205b7b0fc66c1ea91c463fac6334c
            }
            connect_timeout 3
            retry 3
            delay_before_retry 2
        }
    }
}
```

To generate a sum over an URL simply proceed as follows:

```
[root@lvs /root]# genhash –s 192.168.100.2 –p 80 –u /testurl/test.jsp
--------------------------[ HTTP Header Buffer ]--------------------------
0000 48 54 54 50 2f 31 2e 31 - 20 34 30 31 20 55 6e 61 HTTP/1.1 401 Una
0010 75 74 68 6f 72 69 7a 65 - 64 0d 0a 44 61 74 65 3a uthorized..Date:
0020 20 4d 6f 6e 2c 20 32 33 - 20 41 70 72 20 32 30 30 Mon, 23 Apr 200
0030 31 20 31 35 3a 34 31 3a - 35 34 20 47 4d 54 0d 0a 1 15:41:54 GMT..
0040 41 6c 6c 6f 77 3a 20 47 - 45 54 2c 20 48 45 41 44 Allow: GET, HEAD
0050 0d 0a 53 65 72 76 65 72 - 3a 20 4f 72 61 63 6c 65 ..Server: Oracle
0060 5f 57 65 62 5f 4c 69 73 - 74 65 6e 65 72 2f 34 2e _Web_Listener/4.
0070 30 2e 38 2e 31 2e 30 45 - 6e 74 65 72 70 72 69 73 0.8.1.0Enterpris
0080 65 45 64 69 74 69 6f 6e - 0d 0a 43 6f 6e 74 65 6e eEdition..Conten
0090 74 2d 54 79 70 65 3a 20 - 74 65 78 74 2f 68 74 6d t-Type: text/htm
00a0 6c 0d 0a 43 6f 6e 74 65 - 6e 74 2d 4c 65 6e 67 74 l..Content-Lengt
00b0 68 3a 20 31 36 34 0d 0a - 57 57 57 2d 41 75 74 68 h: 164..WWW-Auth
00c0 65 6e 74 69 63 61 74 65 - 3a 20 42 61 73 69 63 20 enticate: Basic
00d0 72 65 61 6c 6d 3d 22 41 - 43 43 45 53 20 20 20 20 realm="ACCES
00e0 22 0d 0a 43 61 63 68 65 - 2d 43 6f 6e 74 72 6f 6c "..Cache-Control
00f0 3a 20 70 75 62 6c 69 63 - 0d 0a 0d 0a : public....
------------------------------[ HTML Buffer ]-----------------------------
0000 3c 48 54 4d 4c 3e 3c 48 - 45 41 44 3e 3c 54 49 54 <HTML><HEAD><TIT
0010 4c 45 3e 55 6e 61 75 74 - 68 6f 72 69 7a 65 64 3c LE>Unauthorized<
0020 2f 54 49 54 4c 45 3e 3c - 2f 48 45 41 44 3e 0d 0a /TITLE></HEAD>..
0030 3c 42 4f 44 59 3e 54 68 - 69 73 20 64 6f 63 75 6d <BODY>This docum
0040 65 6e 74 20 69 73 20 70 - 72 6f 74 65 63 74 65 64 ent is protected
0050 2e 20 20 59 6f 75 20 6d - 75 73 74 20 73 65 6e 64 . You must send
0060 0d 0a 74 68 65 20 70 72 - 6f 70 65 72 20 61 75 74 ..the proper aut
0070 68 6f 72 69 7a 61 74 69 - 6f 6e 20 69 6e 66 6f 72 horization infor
0080 6d 61 74 69 6f 6e 20 74 - 6f 20 61 63 63 65 73 73 mation to access
0090 20 69 74 2e 3c 2f 42 4f - 44 59 3e 3c 2f 48 54 4d it.</BODY></HTM
00a0 4c 3e 0d 0a - L>..
-----------------------[ HTML MD5 final resulting ]-----------------------
MD5 Digest : ec90a42b99ea9a2f5ecbe213ac9eba03
```

The only thing to do is to copy the generated MD5 Digest value generated and paste it into your Keepalived configuration file as a digest value keyword.





# 案例研究：使用VRRP故障转移

As an example we can introduce the following LVS topology:

## Architecture Specification

To create a virtual LVS director using the VRRPv2 protocol, we define the following architecture:

- 2 LVS directors in active-active configuration.
- 4 VRRP Instances per LVS director: 2 VRRP Instance in the MASTER state and 2 in BACKUP state. We use a symmetric state on each LVS directors.
- 2 VRRP Instances in the same state are to be synchronized to define a persistent virtual routing path.
- Strong authentication: IPSEC-AH is used to protect our VRRP advertisements from spoofed and reply attacks.

The VRRP Instances are compounded with the following IP addresses:

- VRRP Instance VI_1: owning VRRIP VIPs VIP1 & VIP2. This instance defaults to the MASTER state on LVS director 1. It stays synchronized with VI_2.
- VRRP Instance VI_2: owning DIP1. This instance is by default in MASTER state on LVS director 1. It stays synchronized with VI_1.
- VRRP Instance VI_3: owning VRRIP VIPs VIP3 & VIP4. This instance is in default MASTER state on LVS director 2. It stays synchronized with VI_4.
- VRRP Instance VI_4: owning DIP2. This instance is in default MASTER state on LVS director 2. It stays synchronized with VI_3.

## Keepalived Configuration

The whole configuration is done in the /etc/keepalived/keepalived.conf file. In our case study this file on LVS director 1 looks like:

```
vrrp_sync_group VG1 {
    group {
        VI_1
        VI_2
    }
}
vrrp_sync_group VG2 {
    group {
        VI_3
        VI_4
    }
}
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 150
    advert_int 1
    authentication {
        auth_type AH
        auth_pass k@l!ve1
    }
    virtual_ipaddress {
        192.168.200.10
        192.168.200.11
    }
}
vrrp_instance VI_2 {
    state MASTER
    interface eth1
    virtual_router_id 52
    priority 150
    advert_int 1
    authentication {
        auth_type AH
        auth_pass k@l!ve2
    }
    virtual_ipaddress {
        192.168.100.10
    }
}
vrrp_instance VI_3 {
    state BACKUP
    interface eth0
    virtual_router_id 53
    priority 100
    advert_int 1
    authentication {
        auth_type AH
        auth_pass k@l!ve3
    }
    virtual_ipaddress {
        192.168.200.12
        192.168.200.13
    }
}
vrrp_instance VI_4 {
    state BACKUP
    interface eth1
    virtual_router_id 54
    priority 100
    advert_int 1
    authentication {
        auth_type AH
        auth_pass k@l!ve4
    }
    virtual_ipaddress {
        192.168.100.11
    }
}
```

Then we define the symmetric configuration file on LVS director 2. This means that VI_3 & VI_4 on LVS director 2 are in MASTER state with a higher priority 150 to start with a stable state. Symmetrically VI_1 & VI_2 on LVS director 2 are in default BACKUP state with lower priority of 100. | This configuration file specifies 2 VRRP Instances per physical NIC. When you run Keepalived on LVS director 1 without running it on LVS director 2, LVS director 1 will own all the VRRP VIP. So if you use the ip utility you may see something like: (On Debian the ip utility is part of iproute):

```
[root@lvs1 tmp]# ip address list
1: lo: <LOOPBACK,UP> mtu 3924 qdisc noqueue
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 brd 127.255.255.255 scope host lo
2: eth0: <BROADCAST,MULTICAST,UP> mtu 1500 qdisc pfifo_fast qlen 100
    link/ether 00:00:5e:00:01:10 brd ff:ff:ff:ff:ff:ff
    inet 192.168.200.5/24 brd 192.168.200.255 scope global eth0
    inet 192.168.200.10/32 scope global eth0
    inet 192.168.200.11/32 scope global eth0
    inet 192.168.200.12/32 scope global eth0
    inet 192.168.200.13/32 scope global eth0
3: eth1: <BROADCAST,MULTICAST,UP> mtu 1500 qdisc pfifo_fast qlen 100
    link/ether 00:00:5e:00:01:32 brd ff:ff:ff:ff:ff:ff
    inet 192.168.100.5/24 brd 192.168.201.255 scope global eth1
    inet 192.168.100.10/32 scope global eth1
    inet 192.168.100.11/32 scope global eth1
```

Then simply start Keepalived on the LVS director 2 and you will see:

```
[root@lvs1 tmp]# ip address list
1: lo: <LOOPBACK,UP> mtu 3924 qdisc noqueue
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 brd 127.255.255.255 scope host lo
2: eth0: <BROADCAST,MULTICAST,UP> mtu 1500 qdisc pfifo_fast qlen 100
    link/ether 00:00:5e:00:01:10 brd ff:ff:ff:ff:ff:ff
    inet 192.168.200.5/24 brd 192.168.200.255 scope global eth0
    inet 192.168.200.10/32 scope global eth0
    inet 192.168.200.11/32 scope global eth0
3: eth1: <BROADCAST,MULTICAST,UP> mtu 1500 qdisc pfifo_fast qlen 100
    link/ether 00:00:5e:00:01:32 brd ff:ff:ff:ff:ff:ff
    inet 192.168.100.5/24 brd 192.168.201.255 scope global eth1
    inet 192.168.100.10/32 scope global eth1
```

Symmetrically on LVS director 2 you will see:

```
[root@lvs2 tmp]# ip address list
1: lo: <LOOPBACK,UP> mtu 3924 qdisc noqueue
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 brd 127.255.255.255 scope host lo
2: eth0: <BROADCAST,MULTICAST,UP> mtu 1500 qdisc pfifo_fast qlen 100
    link/ether 00:00:5e:00:01:10 brd ff:ff:ff:ff:ff:ff
    inet 192.168.200.5/24 brd 192.168.200.255 scope global eth0
    inet 192.168.200.12/32 scope global eth0
    inet 192.168.200.13/32 scope global eth0
3: eth1: <BROADCAST,MULTICAST,UP> mtu 1500 qdisc pfifo_fast qlen 100
    link/ether 00:00:5e:00:01:32 brd ff:ff:ff:ff:ff:ff
    inet 192.168.100.5/24 brd 192.168.201.255 scope global eth1
    inet 192.168.100.11/32 scope global eth1
```

The VRRP VIPs are:

- VIP1 = 192.168.200.10
- VIP2 = 192.168.200.11
- VIP3 = 192.168.200.12
- VIP4 = 192.168.200.13
- DIP1 = 192.168.100.10
- DIP2 = 192.168.100.11

The use of VRRP keyword “sync_instance” imply that we have defined a pair of MASTER VRRP Instance per LVS directors ó (VI_1,VI_2) & (VI_3,VI_4). This means that if eth0 on LVS director 1 fails then VI_1 enters the MASTER state on LVS director 2 so the MASTER Instance distribution on both directors will be: (VI_2) on director 1 & (VI_1,VI_3,VI_4) on director 2. We use “sync_instance” so VI_2 is forced to BACKUP the state on LVS director 1. The final VRRP MASTER instance distribution will be: (none) on LVS director 1 & (VI_1,VI_2,VI_3,VI_4) on LVS director 2. If eth0 on LVS director 1 became available the distribution will transition back to the initial state.

For more details on this state transition please refer to the “Linux Virtual Server High Availability using VRRPv2” paper (available at <http://www.linux-vs.org/~acassen/>), which explains the implementation of this functionality.

Using this configuration both LVS directors are active at a time, thus sharing LVS directors for a global director. That way we introduce a virtual LVS director.

Note

This VRRP configuration sample is an illustration for a high availability router (not LVS specific). It can be used for many more common/simple needs.





# 案例研究：健康检查和故障转移

For this example, we use the same topology used in the Failover part. The idea here is to use VRRP VIPs as LVS VIPs. That way we will introduce a High Available LVS director performing LVS real server pool monitoring.

## Keepalived Configuration

The whole configuration is done in the /etc/keepalived/keepalived.conf file. In our case study this file on LVS director 1 looks like:

```
# Configuration File for keepalived
global_defs {
    notification_email {
        admin@domain.com
        0633225522@domain.com
    }
    notification_email_from keepalived@domain.com
    smtp_server 192.168.200.20
    smtp_connect_timeout 30
    lvs_id LVS_MAIN
}
# VRRP Instances definitions
vrrp_sync_group VG1 {
    group {
        VI_1
        VI_2
    }
}
vrrp_sync_group VG2 {
    group {
        VI_3
        VI_4
    }
}
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 150
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass k@l!ve1
    }
    virtual_ipaddress {
        192.168.200.10
        192.168.200.11
    }
}
vrrp_instance VI_2 {
    state MASTER
    interface eth1
    virtual_router_id 52
    priority 150
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass k@l!ve2
    }
    virtual_ipaddress {
        192.168.100.10
    }
}
vrrp_instance VI_3 {
    state BACKUP
    interface eth0
    virtual_router_id 53
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass k@l!ve3
    }
    virtual_ipaddress {
        192.168.200.12
        192.168.200.13
    }
}
vrrp_instance VI_4 {
    state BACKUP
    interface eth1
    virtual_router_id 54
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass k@l!ve4
    }
    virtual_ipaddress {
        192.168.100.11
    }
}
# Virtual Servers definitions
virtual_server 192.168.200.10 80 {
    delay_loop 30
    lb_algo wrr
    lb_kind NAT
    persistence_timeout 50
    protocol TCP
    sorry_server 192.168.100.100 80
    real_server 192.168.100.2 80 {
        weight 2
        HTTP_GET {
            url {
                path /testurl/test.jsp
                digest ec90a42b99ea9a2f5ecbe213ac9eba03
            }
            url {
                path /testurl2/test.jsp
                digest 640205b7b0fc66c1ea91c463fac6334c
            }
            connect_timeout 3
            retry 3
            delay_before_retry 2
        }
    }
    real_server 192.168.100.3 80 {
        weight 1
        HTTP_GET {
            url {
                path /testurl/test.jsp
                digest 640205b7b0fc66c1ea91c463fac6334c
            }
            connect_timeout 3
            retry 3
            delay_before_retry 2
        }
    }
}
virtual_server 192.168.200.12 443 {
    delay_loop 20
    lb_algo rr
    lb_kind NAT
    persistence_timeout 360
    protocol TCP
    real_server 192.168.100.2 443 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
        }
    }
    real_server 192.168.100.3 443 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
        }
    }
}
```

We define the symmetric VRRP configuration file on LVS director 2. That way both directors are active at a time, director 1 handling HTTP stream and director 2 SSL stream.





# 术语



LVS stands for “Linux Virtual Server“. LVS is a patched Linux kernel that adds a load balancing facility. For more information on LVS, please refer to the project homepage: [http://www.linux-vs.org](http://www.linux-vs.org/). LVS acts as a network bridge (using NAT) to load balance TCP/UDP stream. The LVS router components are:

- WAN Interface: Ethernet Network Interface Controller that will be accessed by all the clients.
- LAN Interface: Ethernet Network Interface Controller to manage all the load balanced servers.
- Linux kernel: The kernel is patched with the latest LVS and is used as a router OS.

In this document, we will use the following keywords:

## LVS Component

- VIP

  The Virtual IP is the IP address that will be accessed by all the clients. The clients only access this IP address.

- Real server

  A real server hosts the application accessed by client requests. WEB SERVER 1 & WEB SERVER 2 in our synopsis.

- Server pool

  A farm of real servers.

- Virtual server

  The access point to a Server pool.

- Virtual Service

  A TCP/UDP service associated with the VIP.

## VRRP Component

- VRRP

  The protocol implemented for the directors’ failover/virtualization.

- IP Address owner

  The VRRP Instance that has the IP address(es) as real interface address(es). This is the VRRP Instance that, when up, will respond to packets addressed to one of these IP address(es) for ICMP, TCP connections, ...

- MASTER state

  VRRP Instance state when it is assuming the responsibility of forwarding packets sent to the IP address(es) associated with the VRRP Instance. This state is illustrated on “Case study: Failover” by a red line.

- BACKUP state

  VRRP Instance state when it is capable of forwarding packets in the event that the current VRRP Instance MASTER fails.

- Real Load Balancer

  An LVS director running one or many VRRP Instances.

- Virtual Load balancer

  A set of Real Load balancers.

- Synchronized Instance

  VRRP Instance with which we want to be synchronized. This provides VRRP Instance monitoring.

- Advertisement

  The name of a simple VRRPv2 packet sent to a set of VRRP Instances while in the MASTER state.

Todo

Define RIP, DIP, Director, IPVS