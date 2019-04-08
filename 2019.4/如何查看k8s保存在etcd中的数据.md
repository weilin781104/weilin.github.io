原文 https://yq.aliyun.com/articles/561888

一直有这个冲动，

想知道kubernetes往etcd里放了哪些数据，是如何组织的。

能看到，才有把握知道它的实现和细节。

找了很多文档，终于找到靠谱的。

===========================

https://www.jianshu.com/p/f9f83dd21770

===========================

 

我是使用kubeadm工具安装的集群，要解除集群的资源占用要先把一些容器停掉，把kube-apiserver的编排文件从/etc/kubernetes/manifests/目录下先移出来，kubelet检查到会停止相应的pods,没有了kube-apiserver集群不会再创建新的pods,这时kubectl不可用了，使用docker命令把spinnaker项目的容器都删掉系统资源就能空闲出来。这时etcd还是正常的，用docker工具直接进入etcd。
操作etcd有命令行工具[etcdctl](https://link.jianshu.com/?t=https%3A%2F%2Fcoreos.com%2Fetcd%2Fdocs%2Flatest%2Fdev-guide%2Flocal_cluster.html)，有两个api版本互不兼容的，系统默认的v2版本，kubernetes集群使用的是v3版本，v2版本下是看不到v3版本的数据的，我也是找了些资料才了解这个情况。
**使用环境变量定义api版本**
`export ETCDCTL_API=3`
**etcd有目录结构类似linux文件系统，获取所有key看一看：**
`etcdctl get / --prefix --keys-only`

**一看就可以大概理解kubenetes的数据结构了，查询命名空间下所有部署的数据：`etcdctl get /registry/deployments/default --prefix --keys-only  `**

**把想删除的删掉，列如：**
`etcdctl del /registry/deployments/default/elevated-dragonfly-spinn-front50`
删除deployments，pods这可以了，稍微减少一些资源，让kube-apiserver可以正常工作即可，其它资源还可以使用kubectl工具删除
删掉些资源后退出etcd把kube-apiserver的编排文件放回/etc/kubernetes/manifests目录，服务会再次启动，然后再清理重新部署。



对于etcd交互特意查询了官方文档（https://coreos.com/etcd/docs/latest/dev-guide/interacting_v3.html）

默认，etcdctl为了后向兼容使用v2 API与etcd server交互。 为了etcdctl使用v3API与etcd Server交互，需要先设置环境变量ETCD_API

```
export ETCDCTL_API=3
```

## 查询版本：

```
$ etcdctl version
etcdctl version: 3.1.0-alpha.0+git
API version: 3.1
```

## 写Key：

```
$ etcdctl put foo bar
OK
```

### 给key一个过期时间

```
$ etcdctl put foo1 bar1 --lease=1234abcd
OK
```

这里1234abcd是一个租期（10s，需要预先建好）

## 读Key：

假设预先写入了一些值：

```
foo = bar
foo1 = bar1
foo2 = bar2
foo3 = bar3
```

### 获取Key foo：

```
$ etcdctl get foo
foo
bar
```

### 获取key foo的16进制

```
$ etcdctl get foo --hex
\x66\x6f\x6f          # Key
\x62\x61\x72          # Value
```

### 只打印值

```
$ etcdctl get foo --print-value-only
bar
```

### 获取范围的key[startkey,endkey)

```
$ etcdctl get foo foo3
foo
bar
foo1
bar1
foo2
bar2
```

### 获取前缀的key

```
$ etcdctl get --prefix foo
foo
bar
foo1
bar1
foo2
bar2
foo3
bar3
```

### 限制获取的个数

```
$ etcdctl get --prefix --limit=2 foo
foo
bar
foo1
bar1
```

### 只打印key

$ etcdctl get --prefix --limit=2 --keys-only foo
foo
foo1

