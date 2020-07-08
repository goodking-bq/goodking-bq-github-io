---
title: "kubernetes 中利用helm安装jenkins"
date: 2020-07-03T10:24:52+08:00
draft: false
tags: ["kubernetes","helm","jenkins"]
categories: ["kubernetes"]
---

##  Jenkins 是什么

是一款开源 CI&CD 软件，用于自动化各种任务，包括构建、测试和部署软件。目前提供超过1000个插件来支持构建、部署、自动化， 满足任何项目的需要。

## 添加helm源

```shell
helm repo add stable https://kubernetes-charts.storage.googleapis.com/
```



## 查看 jenkins 版本

```shell
helm search repo stable/jenkins -l
NAME          	CHART VERSION	APP VERSION	DESCRIPTION
stable/jenkins	2.1.0        	lts        	Open source continuous integration server. It s...
stable/jenkins	2.0.1        	lts        	Open source continuous integration server. It s...
stable/jenkins	2.0.0        	lts        	Open source continuous integration server. It s...
stable/jenkins	1.27.0       	lts        	Open source continuous integration server. It s...
stable/jenkins	1.26.0       	lts        	Open source continuous integration server. It s...
stable/jenkins	1.25.0       	lts        	Open source continuous integration server. It s...
stable/jenkins	1.24.0       	lts        	Open source continuous integration server. It s...
stable/jenkins	1.23.2       	lts        	Open source continuous integration server. It s...
stable/jenkins	1.23.1       	lts        	Open source continuous integration server. It s...
stable/jenkins	1.23.0       	lts        	Open source continuous integration server. It s...
stable/jenkins	1.22.0       	lts        	Open source continuous integration server. It s...
stable/jenkins	1.21.3       	lts        	Open source continuous integration server. It s...
stable/jenkins	1.21.2       	lts        	Open source continuous integration server. It s...
stable/jenkins	1.21.1       	lts        	Open source continuous integration server. It s...
stable/jenkins	1.21.0       	lts        	Open source continuous integration server. It s...
stable/jenkins	1.20.0       	lts        	Open source continuous integration server. It s...
... 非常多
```

## 安装最新版

```sh
# -n 设置namespace  
# --set persistence.storageClass 设置存储
helm install jenkins stable/jenkins -n kube-system --set persistence.storageClass=nfs-storage
```

或是

```sh
helm fetch stable/jenkins --untar
cd jenkins
# 编辑配置
vim values.yaml

```

> values.yaml 配置说明可以查看 [查看说明](https://hub.helm.sh/charts/stable/jenkins)



默认安装了一些插件：

 - kubernetes
 - workflow-job
 - workflow-aggregator
 - credentials-binding
 - git
 - configuration-as-code

## 设置中文

1. 安装 **Locale** 插件
2. 重启jenkins
3. 配置【Manage Jenkins】>【Configure System】> 【Locale】设置为 **zh_CN**

## 遇到的问题 

1. 在装插件的时候一直报错，**Failed to download plugin: xxx** ，通过  *kubectl logs jenkins-fc4c8755c-hkqkf copy-default-config* 可以看到日志，

   可以通过设置 **master.hostNetworking=true** 解决了。

2. 在gitlab上出发webhook构建时报错：**Error 403 No valid crumb was included in the request**

   由于新版不能关掉csrf

