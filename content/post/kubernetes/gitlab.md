---
title: "Kubernetes 集群安装 Gitlab "
date: 2020-07-08T10:57:44+08:00
draft: false
tags: ["kubernetes","how-to"]
categories: ["kubernetes"]
---

## helm添加gitlab源

```shell
helm repo add gitlab https://charts.gitlab.io/
```

## 安装

```shell
helm fetch gitlab/gitlab --untar
cp gitlab/values.yaml gitlab-config.yaml
```

编辑 **gitlab-config.yaml**：

仅给出了我自己的修改

```yaml
global:
	edition: ce # ce版本
	hosts：
		domain: xxxxx # 访问的地址
	gitlab: 
    persistence: # 设置存储
      storageClass: nfs-storage
      enabled: true
      accessMode: ReadWriteOnce
  psql: # psql 我搭建在物理机上的。
    password:
      secret: gitlab-gitlab-ce
      key: db-password
    host: 192.168.238.253
    # port: 123
    username: yunwei
    database: gitlab
	gitaly: # gitaly 存储
    persistence:
      storageClass: nfs-storage
      enabled: true
      accessMode: ReadWriteOnce
	minio: # minio 存储
    enabled: true
    persistence:
      storageClass: nfs-storage
      enabled: true
      accessMode: ReadWriteOnce
  grafana:
    enabled: false # 自己搭建了 不需要
  incomingEmail:
      enabled: false 
  serviceDeskEmail: # 邮箱配置根据自己需要
      enabled: false
  geo:
    enabled: false # 不需要
  registry:
    enable: false # 已经搭建了harbor了 不需要它的镜像服务
 certmanager-issuer:
#   # The email address to register certificates requested from Let's Encrypt.
#   # Required if using Let's Encrypt.
  email: xxx # ！这个必填
nginx-ingress:
  enabled: false # 我用的traefik，不需要它
prometheus:
  install: false # 自己搭了 不需要
postgresql:
	install: false # 自己搭了 不需要
gitlab-runner:
  install: false # 使用 helm 仓库 gitlab/gitlab-runner 自己搭吧 可配置性更高些
```

现在还不能直接 **helm install -f gitlab-config.yaml gitlab gitlab** ,因为它的镜像差不多都会下载失败。

```sh
helm template -f gitlab-config.yaml gitlab gitlab >gitlab.yaml
cat gitlab.yaml|grep image:
					image: "quay.io/jetstack/cert-manager-cainjector:v0.10.1"
          image: "quay.io/jetstack/cert-manager-controller:v0.10.1"
          image: "registry.gitlab.com/gitlab-org/build/cng/alpine-certificates:20171114-r3"
          image: "busybox:latest"
          image: "registry.gitlab.com/gitlab-org/build/cng/gitlab-exporter:7.0.6"
          image: "registry.gitlab.com/gitlab-org/build/cng/alpine-certificates:20171114-r3"
          image: "busybox:latest"
          image: "registry.gitlab.com/gitlab-org/build/cng/gitlab-shell:v13.3.0"
          image: "registry.gitlab.com/gitlab-org/build/cng/alpine-certificates:20171114-r3"
          image: "busybox:latest"
          image: "registry.gitlab.com/gitlab-org/build/cng/gitlab-sidekiq-ce:v13.1.2"
          image: "registry.gitlab.com/gitlab-org/build/cng/gitlab-sidekiq-ce:v13.1.2"
          image: "registry.gitlab.com/gitlab-org/build/cng/alpine-certificates:20171114-r3"
          image: "busybox:latest"
          image: "registry.gitlab.com/gitlab-org/build/cng/gitlab-task-runner-ce:v13.1.2"
          image: "registry.gitlab.com/gitlab-org/build/cng/alpine-certificates:20171114-r3"
          image: "busybox:latest"
          image: registry.gitlab.com/gitlab-org/build/cng/gitlab-webservice-ce:v13.1.2
          image: registry.gitlab.com/gitlab-org/build/cng/gitlab-webservice-ce:v13.1.2
          image: "registry.gitlab.com/gitlab-org/build/cng/gitlab-workhorse-ce:v13.1.2"
          image: "busybox:latest"
          image: minio/minio:RELEASE.2017-12-28T01-21-00Z
          image: "registry.gitlab.com/gitlab-org/build/cng/alpine-certificates:20171114-r3"
          image: "busybox:latest"
          image: "registry.gitlab.com/gitlab-org/build/cng/gitaly:v13.1.2"
        image: "docker.io/bitnami/redis:5.0.7-debian-9-r50"
        image: docker.io/bitnami/redis-exporter:1.3.5-debian-9-r23
          image: "registry.gitlab.com/gitlab-org/build/cng/alpine-certificates:20171114-r3"
          image: "busybox:latest"
          image: "registry.gitlab.com/gitlab-org/build/cng/gitlab-task-runner-ce:v13.1.2"
        image: minio/mc:RELEASE.2018-07-13T00-53-22Z
    image: registry.gitlab.com/gitlab-org/build/cng/gitlab-webservice-ce:v13.1.2
          image: "registry.gitlab.com/gitlab-org/build/cng/kubectl:1.13.12"
          image: "busybox:latest"
```

你需要翻墙下下来上传到国内镜像源，如阿里，或是去阿里找找。 **registry.gitlab.com，quay.io **的都需要。

改后 **kubectl apply -f gitlab.yaml** 就成功安装了。

## gitlab 的 traefik 配置

```sh
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: gitlab-https
spec:
  entryPoints:
    - websecure
  routes:
  - match: Host(`gitlab.***.com`) && PathPrefix(`/`)
    kind: Rule
    services:
    - name:  gitlab-webservice
      port: 8181
  - match: Host(`gitlab.***.com`) && PathPrefix(`/admin/sidekiq`)
    kind: Rule
    services:
    - name:  gitlab-webservice
      port: 8080
  tls:
    certResolver: default
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: minio-https
spec:
  entryPoints:
    - websecure
  routes:
  - match: Host(`minio.***.com`)
    kind: Rule
    services:
    - name:  gitlab-minio-svc
      port: 9000
  tls:
    certResolver: default
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: gitlab-http
spec:
  entryPoints:
    - web
  routes:
  - match: Host(`gitlab.***.com`)
    kind: Rule
    #    middlewares:
    #- name: http-to-https
    services:
    - name: gitlab-webservice
      port: 8181
```

这就全部完成了。

## 获取管理员密码

> 管理员用户名：root
>
> 管理员用户邮箱： admin@example.com

获取默认密码

```sh
kubectl get secret gitlab-gitlab-initial-root-password -ojsonpath='{.data.password}' | base64 --decode ;echo
PgU8ckN6os9nK0oeCXKbaE6vjKV6p7gmYNfGHhnn6Du5bdUW8Fn5q69zwQr1Q1tp # 这就是管理员密码
```

## 设置管理员密码

如果不对，那么只有重新设置了。

```sh
kubectl get pod |grep task-runner # 找到pod  gitlab-task-runner-6f64c9566f-bmmsj
kubectl exec -it gitlab-task-runner-6f64c9566f-bmmsj -- /srv/gitlab/bin/rails console
--------------------------------------------------------------------------------
 GitLab:       13.1.2 () FOSS
 GitLab Shell: 13.3.0
 PostgreSQL:   12.2
--------------------------------------------------------------------------------

user = User.where(id: 1).first
user.password = 'secret_pass'
user.password_confirmation = 'secret_pass'

user.save!

```

然后用 admin@example.com + 新密码就可以了。

