---
title: "kubectl kustomize 使用"
date: 2020-06-01T15:08:53+08:00
draft: true
tags: ["kubernetes","how-to"]
categories: ["kubernetes"]
---
> kubectl v1.14及以后版本都支持 ，1.14以前的需要下载二进制文件安装
> 
> kustomize 是kubernetes原生的配置管理工具。
> 
> 它无需模板即可定制kubernetes配置，简单方便， 
> 
> 通过**kubectl kustomize dir**来输出配置或是 **kubectl apply -k dir**来直接使用配置

