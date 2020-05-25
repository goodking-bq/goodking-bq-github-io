---
title: "kubernetes 的 configmap 使用"
date: 2020-05-15T10:11:48+08:00
draft: false
tags: ["kubernetes","how-to"]
categories: ["kubernetes"]
---

## kubernetes 的 configmap

> ConfigMap 是 configMap 是一种 API 对象，用来将非机密性的数据保存到健值对中。使用时可以用作环境变量、命令行参数或者存储卷中的配置文件。

> ConfigMap 将您的环境配置信息和 容器镜像 解耦，便于应用配置的修改。

> 它是用来存放应用配置的。

> ConfigMap 只能被同一个命名空间里的 Pod 所引用
## 如何创建

> 它不像其他对象有spec，它用的data来存储建值

你可以使用命令来创建：
```shell
# 使用字面值
kubectl create configmap test-config --from-literral=a=1 --from-literral=v.k=1
# 使用文件创建
kubectl create configmap test-config --from-file="/data/config.py"
kubectl create configmap test-config --from-file=config-key="/data/config.py"
# 使用目录创建
kubectl create configmap test-config --from-file="/data/config" 
# 
cat <<EOF >> cm.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  Name: game-demo
  namespace: default
data:
  # 类属性键；每一个键都映射到一个简单的值
  player_initial_lives: 3
  ui_properties_file_name: "user-interface.properties"
  #
  # 类文件键
  game.properties: |
    enemy.types=aliens,monsters
    player.maximum-lives=5
  user-interface.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true
EOF

kubectl apply -f cm.yaml
```

## 如何使用

### 环境变量的方式

>  将配置注入到pod环境变量中，通过读取环境变量的值来使用它

如何在pod中使用configmap来定义环境变量:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: gcr.io/google_containers/busybox
      command: [ "/bin/sh", "-c", "env" ]
      env:
        # 定义环境变量名称
        - name: SPECIAL_LEVEL_KEY
          # 值从哪里来
          valueFrom:
            # 值从configmap 来
            configMapKeyRef:
              # configmap的名字
              name: special-config
              # configmap的key
              key: special.how
      # 将configmap的所有key-value 设置成环境变量
      envFrom:
      - configMapRef:
          # configmap的名字
          name: special-config
  restartPolicy: Never
```
### 挂载卷的方式

> 将 configmap 通过挂载卷挂载到pod系统中，程序通过读取系统的配置文件来使用

如何挂载到卷中呢,官方给的例子：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: gcr.io/google_containers/busybox
      command: [ "/bin/sh", "-c", "ls /etc/config/" ]
      # 使用
      volumeMounts:
      - name: config-volume
        # 挂载路径
        mountPath: /etc/config
        # 只读权限
        readOnly: true 
  volumes:
    - name: config-volume
      # 挂载整个configmap
      configMap:
        # configmap的名字
        name: special-config
    - name: config-volume-1
      # 挂载某个值
      configMap:
        name: special-config
        items:
        - key: special.level
          path: keys
  restartPolicy: Never
```

## 关于自动更新

> 当一个已经被使用的 ConfigMap 发生了更新时，对应的 Key 也会被更新。Kubelet 会周期性的检查挂载的 ConfigMap 是否是最新的。 然而，它会使用本地基于 ttl 的 cache 来获取 ConfigMap 的当前内容。因此，从 ConfigMap 更新到 Pod 里的新 Key 更新这个时间，等于 Kubelet 的同步周期加 ConfigMap cache 的 tll。