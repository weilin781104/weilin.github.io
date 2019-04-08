

stripPrefix默认true，会自动把前缀去掉，代理请求发到应用，它会把去掉的前缀放到header字段x-forwarded-prefix。 如果配置stripPrefix为false，就是不去掉前缀，也就没有header字段x-forwarded-prefix



## 默认stripPrefix为true

### 配置：

```
zuul:
  routes:
    booting:
      path: /booting*/**
      serviceId: iptv3a-booting
```

### 查看配置生效：

![](assets\20190408214502.png)

### 访问：
curl http://192.168.11.89:8501/booting1/88880001/login

### 应用侧抓包：

![](assets\20190408214316.png)


### 备注
这里有一个要注意的地方：如果prefix里带了通配符，虽然能正确去掉prefix，url正确（这里是/88880001/login），但是x-forwarded-prefix是不对的，看前面配置生效查看里的path和prefix也是不对的



## stripPrefix设置为false

### 配置：

```
zuul:
  routes:
    booting:
      path: /booting*/**
      serviceId: iptv3a-booting
      stripPrefix: false
```

### 查看配置生效

![img](assets\R[WSSH6EO7ME]ZNV}XOSJW.png)



### 访问：
curl http://192.168.11.89:8501/booting1/88880001/login

### 应用侧抓包：

![](assets\20190408214807.png)

## 没有全局配置

zuul.stripPrefix`只对zuul.prefix配置的前缀生效，不是全局的，不会对特定路由生效。

```
zuul:
  prefix: api
  stripPrefix: false
  routes:
    users:
      path: /myusers/**
      serviceId: users
```

这里对users路由来说还是默认的去prefix


## 结论
对于没有显性加prefix的情况，一定要保留prefix才能正确路由到后端应用，没有全局配置的情况下，每个routes都要配置stripPrefix=false