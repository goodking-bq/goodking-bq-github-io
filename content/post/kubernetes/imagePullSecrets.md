---
title: "配置 kubernetes 拉取私有仓库"
date: 2020-05-25T15:26:54+08:00
draft: false
tags: ["kubernetes","how-to"]
categories: ["kubernetes"]
---

docker拉取镜像是不需要登陆的，那怎么拉取哪些需要认证的私密镜像呢？

可以使用**secret**

## 生成 **secret**

```shell
kubectl create secret docker-registry aliyun-auth --docker-server=registry.cn-hangzhou.aliyuncs.com --docker-username=avc@qq.com --docker-password=121212 
```
最后生成的数据类似这样的：
```json
{"auths":{"registry.cn-hangzhou.aliyuncs.com":{"username":"avc@qq.com","password":"121212","auth":"MTIwMjI1ODgzQHFxLmNvbTp6MTIwMaaaaaaa"}}}
```

> 注意 只有同名称空间才能使用secret

## 使用 **secret** 拉取镜像

> 通过 **imagePullSecrets** 字段配置

```yaml
containers:
- name: channel
  image: registry-internal.cn-hangzhou.aliyuncs.com/yourname/redis
# 使用 aliyun
imagePullSecrets:
- name: aliyun
```

