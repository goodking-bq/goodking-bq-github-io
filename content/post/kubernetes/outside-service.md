---
title: "kubernetes 集群内部应用访问集群外部的服务"
date: 2020-06-01T15:58:04+08:00
draft: false
tags: ["kubernetes","how-to"]
categories: ["kubernetes"]
---

我们经常需要 kubernetes 集群内部应用访问集群外部的服务，
比如mysql启动在集群外。有两种方式。

## Service type:ExternalName

> 集群内不访问通常都是通过dns访问，也就是 **service**。**type=ExternalName**的服务能将外部的一个域名映射为集群内部的一个服务

> 但是这种方式不能指定端口

```yaml
kind: Service
apiVersion: v1
metadata:
  name: svc1
  namespace: default
spec:
  type: ExternalName
  externalName: outside-domain.com
```
上面例子就是将 **outside-domain.com** 映射为 **svc1**服务，集群内访问svc1就行了。这里没有端口 。注意！！

## Service 的Endpoint 

> Endpoint 资源暴露一个服务器的地址和端口给 Service，这个地址可以是集群内部的，也可以是集群外部的地址

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-production
spec:
  ports:
    - port: 3306
---
kind: Endpoints
apiVersion: v1
metadata:
  name: mysql-production
  namespace: default
subsets:
  - addresses:
      - ip: 192.168.11.11
    ports:
      - port: 3306
```

上面定义的 **Service** 和 **Endpoints**是同名的。

