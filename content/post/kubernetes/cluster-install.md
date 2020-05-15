---
title: "kubernetes master集群搭建"
date: 2020-05-15T10:24:52+08:00
draft: false
---
##  开始之前

kubernetes master 集群有两种模式

- 堆叠（Stacked） etcd 拓扑
  每个节点都运行 kube-apiserver，kube-scheduler ， kube-controller-manager， etcd实例。
  
  kube-apiserver 使用负载均衡器暴露给工作节点,每个控制平面节点创建一个本地 etcd 成员（member），这个 etcd 成员只与该节点的 kube-apiserver 通信。这同样适用于本地 kube-controller-manager 和 kube-scheduler 实例。

  这是 kubeadm 中的默认拓扑。当使用 kubeadm init 和 kubeadm join --control-plane 时，在控制平面节点上会自动创建本地 etcd 成员。
  ![堆叠（Stacked） etcd 拓扑](/kubernetes/kubeadm-ha-topology-stacked-etcd.svg)

- 外部 etcd 拓扑
  
  etcd单独的集群，每个etcd与控制节点通过apiserver通信，这种拓扑结构解耦了控制平面和 etcd 成员。因此，它提供了一种 HA 设置，其中失去控制平面实例或者 etcd 成员的影响较小，并且不会像堆叠的 HA 拓扑那样影响集群冗余。

  但是，此拓扑需要两倍于堆叠 HA 拓扑的主机数量。



  ![外部 etcd 拓扑](/kubernetes/kubeadm-ha-topology-external-etcd.svg)

## 服务器

| hostname | 内网地址        | 角色   | 系统               | 用户 |
| -------- | --------------- | ------ | ------------------ | ---- |
| master1  | 192.168.192.231 | master | ubuntu focal 20.04 | root |
| master2  | 192.168.238.255 | master | ubuntu focal 20.04 | root |
| master3  | 192.168.238.167 | master | ubuntu focal 20.04 | root |



## 准备环境

### - 设置hostname 以及hosts

按上面的配置更改每台的 **hostname** 以及 **/etc/hosts**

```sh
hostnamectl set-hostname master2
cat <<EOF >> /etc/hosts
192.168.192.231 master1
192.168.238.167 master3
192.168.238.255 master2
EOF
```

### - 保证每台互相之间SSH无密码互通

master1上执行
```sh
ssh-keygen -t rsa -P ""
cd .ssh
scp id_rsa*  master2:.ssh
scp id_rsa*  master3:.ssh
```

将.ssh/id_rsa.pub .ssh/id_rsa复制到其他两台相同的目录执行相同，每台执行

```sh
cat .ssh/id_rsa.pub>>.ssh/authorized_keys
```



### - 在每台服务器上安装docker

- 添加源，focal 太新了还没得源，所以用了 xenial 源
```sh
cat <<EOF > /etc/apt/sources.list.d/docker.list
deb [arch=amd64] https://mirrors.aliyun.com/docker-ce/linux/ubuntu Bionic stable
EOF
```

- 安装
```sh
sudo apt update && sudo apt install containerd docker.io
```

- 设置自动启动
```sh
sudo systemctl enable docker
sudo systemctl start docker
```

如果提示 ```Failed to enable unit: Unit file /etc/systemd/system/docker.service is masked.```
 
执行 
```shell
sudo systemctl unmask docker.socket
sudo systemctl unmask docker.service
```

### - 在每台服务器上安装kubernetes

- 添加源，focal 太新了还没得源，所以用了 xenial 源
    ```sh
    cat <<EOF > /etc/apt/sources.list.d/kubernetes.list
    deb http://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial main
    EOF
    sudo apt update
    ```
    如果update提示
    ``` 
    W: GPG error: http://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial InRelease: The following signatures couldn't be verified because the public key is not available: NO_PUBKEY 6A030B21BA07F4FB
    E: The repository 'http://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial InRelease' is not signed.
    N: Updating from such a repository can't be done securely, and is therefore disabled by default.
    N: See apt-secure(8) manpage for repository creation and user configuration details.
    ```
    只需要添加pubkey就可以了 我这里是 **NO_PUBKEY 6A030B21BA07F4FB**，最后8位 **BA07F4FB**
    执行下面命令：
    ```shell
    gpg --keyserver keyserver.ubuntu.com --recv-keys BA07F4FB
    gpg --export --armor BA07F4FB | sudo apt-key add -
    sudo apt-get update
    ```

- 安装kube
  
    ```sh
    sudo apt-get install -y kubelet kubeadm kubectl
    sudo apt-mark hold kubelet kubeadm kubectl
    ```  
- 关闭swap
  
    ```sh
    sudo swapoff -a
    ```
    要永久生效只需要把 **/etc/fstab**中的swap列注释掉


### - 安装etcd集群(可选，这是针对单独的etcd集群，)

参考官方文档 [使用 kubeadm 创建一个高可用 etcd 集群](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/setup-ha-etcd-with-kubeadm/)

- 在每台上执行
```sh
sudo apt install etcd -y
```

- 将 kubelet 配置为 etcd 的服务管理器。 在每台上执行
```sh
cat << EOF > /etc/systemd/system/kubelet.service.d/20-etcd-service-manager.conf
[Service]
ExecStart=
#  Replace "systemd" with the cgroup driver of your container runtime. The default value in the kubelet is "cgroupfs".
ExecStart=/usr/bin/kubelet --address=127.0.0.1 --pod-manifest-path=/etc/kubernetes/manifests --cgroup-driver=systemd
Restart=always
EOF

systemctl daemon-reload
systemctl restart kubelet
```
- 为 kubeadm 创建配置文件。
```sh

# 使用 IP 或可解析的主机名替换 HOST0、HOST1 和 HOST2
export HOST0=master1
export HOST1=master2
export HOST2=master3
# 使用了hosts
# 创建临时目录来存储将被分发到其它主机上的文件
mkdir -p /tmp/${HOST0}/ /tmp/${HOST1}/ /tmp/${HOST2}/

ETCDHOSTS=(${HOST0} ${HOST1} ${HOST2})
NAMES=("infra0" "infra1" "infra2")

for i in "${!ETCDHOSTS[@]}"; do
HOST=${ETCDHOSTS[$i]}
NAME=${NAMES[$i]}
cat << EOF > /tmp/${HOST}/kubeadmcfg.yaml
apiVersion: "kubeadm.k8s.io/v1beta2"
kind: ClusterConfiguration
etcd:
    local:
        serverCertSANs:
        - "${HOST}"
        peerCertSANs:
        - "${HOST}"
        extraArgs:
            initial-cluster: infra0=https://${ETCDHOSTS[0]}:2380,infra1=https://${ETCDHOSTS[1]}:2380,infra2=https://${ETCDHOSTS[2]}:2380
            initial-cluster-state: new
            name: ${NAME}
            listen-peer-urls: https://${HOST}:2380
            listen-client-urls: https://${HOST}:2379
            advertise-client-urls: https://${HOST}:2379
            initial-advertise-peer-urls: https://${HOST}:2380
EOF
done
```
- 生成证书颁发机构
```sh
kubeadm init phase certs etcd-ca
```

-  为每个成员创建证书
```sh
kubeadm init phase certs etcd-server --config=/tmp/${HOST2}/kubeadmcfg.yaml
kubeadm init phase certs etcd-peer --config=/tmp/${HOST2}/kubeadmcfg.yaml
kubeadm init phase certs etcd-healthcheck-client --config=/tmp/${HOST2}/kubeadmcfg.yaml
kubeadm init phase certs apiserver-etcd-client --config=/tmp/${HOST2}/kubeadmcfg.yaml
cp -R /etc/kubernetes/pki /tmp/${HOST2}/
# 清理不可重复使用的证书
find /etc/kubernetes/pki -not -name ca.crt -not -name ca.key -type f -delete

kubeadm init phase certs etcd-server --config=/tmp/${HOST1}/kubeadmcfg.yaml
kubeadm init phase certs etcd-peer --config=/tmp/${HOST1}/kubeadmcfg.yaml
kubeadm init phase certs etcd-healthcheck-client --config=/tmp/${HOST1}/kubeadmcfg.yaml
kubeadm init phase certs apiserver-etcd-client --config=/tmp/${HOST1}/kubeadmcfg.yaml
cp -R /etc/kubernetes/pki /tmp/${HOST1}/
find /etc/kubernetes/pki -not -name ca.crt -not -name ca.key -type f -delete

kubeadm init phase certs etcd-server --config=/tmp/${HOST0}/kubeadmcfg.yaml
kubeadm init phase certs etcd-peer --config=/tmp/${HOST0}/kubeadmcfg.yaml
kubeadm init phase certs etcd-healthcheck-client --config=/tmp/${HOST0}/kubeadmcfg.yaml
kubeadm init phase certs apiserver-etcd-client --config=/tmp/${HOST0}/kubeadmcfg.yaml
# 不需要移动 certs 因为它们是给 HOST0 使用的

# 清理不应从此主机复制的证书
find /tmp/${HOST2} -name ca.key -type f -delete
find /tmp/${HOST1} -name ca.key -type f -delete
```

- 复制证书到其他两台上
```sh
scp -r /tmp/master2/* master2:
scp -r /tmp/master3/* master3:
ssh master2 "mv pki /etc/kubernetes/"
ssh master3 "mv pki /etc/kubernetes/"
```
检查下文件
master1:
```
/tmp/${HOST0}
└── kubeadmcfg.yaml
---
/etc/kubernetes/pki
├── apiserver-etcd-client.crt
├── apiserver-etcd-client.key
└── etcd
    ├── ca.crt
    ├── ca.key
    ├── healthcheck-client.crt
    ├── healthcheck-client.key
    ├── peer.crt
    ├── peer.key
    ├── server.crt
    └── server.key
```
master2,master3:
```
$HOME
└── kubeadmcfg.yaml
---
/etc/kubernetes/pki
├── apiserver-etcd-client.crt
├── apiserver-etcd-client.key
└── etcd
    ├── ca.crt
    ├── healthcheck-client.crt
    ├── healthcheck-client.key
    ├── peer.crt
    ├── peer.key
    ├── server.crt
    └── server.key
```

- 创建pod
在mater1执行
```
kubeadm init phase etcd local --config=/tmp/master1/kubeadmcfg.yaml
ssh master2 "kubeadm init phase etcd local --config=/home/ubuntu/kubeadmcfg.yaml"
ssh master3 "kubeadm init phase etcd local --config=/home/ubuntu/kubeadmcfg.yaml"
```