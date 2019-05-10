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



|      |      |      |
| ---- | ---- | ---- |
|      |      |      |
|      |      |      |
|      |      |      |
|      |      |      |
|      |      |      |
|      |      |      |
|      |      |      |
|      |      |      |
|      |      |      |
|      |      |      |
|      |      |      |
|      |      |      |
|      |      |      |
|      |      |      |
|      |      |      |
|      |      |      |
|      |      |      |

