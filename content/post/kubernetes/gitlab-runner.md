---
title: "kubernetes Gitlab Runner 的安装"
date: 2020-07-14T22:30:13+08:00
draft: false
tags: ["kubernetes","how-to", "gitlab runner"]
categories: ["kubernetes"]
---

## helm 添加 gitlab源

```sh
helm repo add gitlab https://charts.gitlab.io/
```

## 下载及安装

### 1. 下载

```sh
helm fetch gitlab/gitlab-runner --untar
cp gitlab/values.yaml gitlab-runner-config.yaml
```

### 2. 编辑配置文件

一些配置及说明

```yaml
# gitlab 地址
gitlabUrl: https://gitlab.xxx.com
# ! 非常重要，需要从 gitlab > 管理中心 > runner 页面获取令牌
# 注册时需要
runnerRegistrationToken: "xxx"
rbac:
  create: true
  ## Define specific rbac permissions.
  resources: ["pods", "pods/exec", "secrets"]
  verbs: ["get", "list", "watch", "create", "patch", "delete"]

  ## Run the gitlab-bastion container with the ability to deploy/manage containers of jobs
  ## cluster-wide or only within namespace
  clusterWideAccess: true
# 监控需要不
metrics:
  enabled: false
runners:
  ## Default container image to use for builds when none is specified
  ##
  image: ubuntu:20.04
  # 这个设置为false吧，不锁定这个runner，就是所有项目公用
  locked: false
  # 非常需要，dockers in docker时必须设置为true才行
  privileged: true
  # 执行的container性能配置，直接影响自动化执行的速度
  builds:
    cpuLimit: 200m
    cpuLimitOverwriteMaxAllowed: 400m
    memoryLimit: 256Mi
    memoryLimitOverwriteMaxAllowed: 512Mi
    cpuRequests: 100m
    cpuRequestsOverwriteMaxAllowed: 200m
    memoryRequests: 128Mi
    memoryRequestsOverwriteMaxAllowed: 256Mi

  ## Service Container specific configuration
  ## 服务的container
  ## 需要用docker打包镜像时需要。直接影响docker build速度
  services:
    cpuLimit: 2000m
    memoryLimit: 2Gi
    cpuRequests: 1000m
    memoryRequests: 2Gi

  ## Helper Container specific configuration
  ##
  helpers:
    cpuLimit: 200m
    memoryLimit: 256Mi
    cpuRequests: 100m
    memoryRequests: 128Mi
  # 使用docker in docker时需要配置的参数，这里写了就不需要在ci配置文件里面写了
  env:
    DOCKER_HOST: tcp://localhost:2375
    DOCKER_TLS_CERTDIR: ""
    DOCKER_DRIVER: overlay2
  envVars:
  - name: RUNNER_EXECUTOR
    value: kubernetes ## kubernetes
```

### 3. 安装

```sh
helm install -f gitlab-runner-config.yaml gitlab-runner gitlab-runner
```

### 4. 检查gitlab runner是否注册了。