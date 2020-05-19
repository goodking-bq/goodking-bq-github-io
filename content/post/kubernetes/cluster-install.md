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

> 无论哪一种，master集群都需要一个外部的负载均衡器，这里我用了haproxy+keepalived做的软负载均衡

**我这里做的是第二种，外部tecd拓扑**

## 服务器说明
这里只有三太服务器所有的都装一起了。

| hostname | 内网地址        | 角色      | 系统               | 用户 |
| -------- | --------------- | --------- | ------------------ | ---- |
| master1  | 192.168.192.231 | master,hk | ubuntu focal 20.04 | root |
| master2  | 192.168.238.255 | master,hk | ubuntu focal 20.04 | root |
| master3  | 192.168.238.167 | master,hk | ubuntu focal 20.04 | root |
|          | 192.168.255.254 |           | 负载均衡地址       |      |

说明：

- hk: haproxy+keepalived
- master: kubernetes control plane , apiserver+ controller-manster+scheduler
- etcd: etcd cluster node

## 准备环境

### 设置hostname 以及hosts

按上面的配置更改每台的 **hostname** 以及 **/etc/hosts**

```sh
hostnamectl set-hostname master2
cat <<EOF >> /etc/hosts
192.168.192.231 master1
192.168.238.167 master3
192.168.238.255 master2
EOF
```

###  保证每台互相之间SSH无密码互通

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



###  在每台服务器上安装docker

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

###  在每台服务器上安装kubernetes

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

## 安装设置负载均衡 keepalived + haproxy

> - keepalived 可以使多台机器组成高可用，提供一个漂移 IP，对外提供服务，它有两种模式： 抢占式、非抢占式 ，这里使用的是抢占式，非抢占请自行google
> - haproxy HAProxy 是一款提供高可用性、负载均衡以及基于TCP（第四层）和HTTP（第七层）应用的代理软件，支持虚拟主机，它是免费、快速并且可靠的一种解决方案

- 命令安装，执行
```shell
sudo apt install keepalived haproxy
```
- 设置keepalived
找一个内网没被使用的ip： 192.168.255.254
```sh
cat << EOF > /etc/keepalived/keepalived.conf
! Configuration File for keepalived

global_defs {
   router_id LVS_DEVEL_MASTER2 # 名字，每台不一样
}

vrrp_script check_haproxy {
    script "killall -0 haproxy"
    interval 3
    weight -2
    fall 10
    rise 2
}

vrrp_instance VI_1 {
    state MASTER # 这里有两种 MASTER|BACKUP,MASTER挂了会自动切换到BACKUP 
    interface eno2 # 绑定的网卡，不要学错了。
    virtual_router_id 51 # 虚拟路由id，集群里必须一样
    priority 150 # 切换权重
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 35f18af7190d51c9f7f78f37300a0cbd
    }
    virtual_ipaddress {
        192.168.255.254  # 重要，设置负载均衡的ip
    }
    track_script {
        check_haproxy # 检查脚本
    }
}
EOF 
systemctl restart keepalived.service
```
> 检查MASTER节点的网卡上是不是多了个ip ： 192.168.255.254

- 设置haproxy

```shell
cat << EOF > /etc/keepalived/keepalived.conf

# 全局配置
global
	log /dev/log	local0
	log /dev/log	local1 notice
	chroot /var/lib/haproxy
	stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
	stats timeout 30s
	user haproxy
	group haproxy
	daemon

	# Default SSL material locations
	ca-base /etc/ssl/certs
	crt-base /etc/ssl/private

	# See: https://ssl-config.mozilla.org/#server=haproxy&server-version=2.0.3&config=intermediate
        ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
        ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
        ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets

defaults
	log	global
	mode	http
	option	httplog
	option	dontlognull
        timeout connect 5000
        timeout client  50000
        timeout server  50000
	errorfile 400 /etc/haproxy/errors/400.http
	errorfile 403 /etc/haproxy/errors/403.http
	errorfile 408 /etc/haproxy/errors/408.http
	errorfile 500 /etc/haproxy/errors/500.http
	errorfile 502 /etc/haproxy/errors/502.http
	errorfile 503 /etc/haproxy/errors/503.http
	errorfile 504 /etc/haproxy/errors/504.http

#---------------------------------------------------------------------
#  frontend for api
#---------------------------------------------------------------------
frontend kubernetes-apiserver
    mode                 tcp
    bind                 *:16443
    option               tcplog
    default_backend      kubernetes-apiserver

#---------------------------------------------------------------------
# backend for api
#---------------------------------------------------------------------
backend kubernetes-apiserver
    mode        tcp
    balance     roundrobin
    server  master-0 master1:6443 check
    server  master-1 master2:6443 check
    server  master-2 master3:6443 check

#---------------------------------------------------------------------
# 状态监控
#---------------------------------------------------------------------
listen stats
    bind                 *:1080
    stats auth           admin:awesomePassword
    stats refresh        5s
    stats realm          HAProxy\ Statistics
    stats uri            /admin?stats
EOF 
sudo systemctl restart haproxy
```

## 单独安装etcd集群(可选，这是针对单独的etcd集群，)

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
上面的命令是在 **/etc/kubernetes# cd manifests/** 下面创建etcd.yaml,当kubelet启动时，会创建docker etcd,我这里想让etcd不在docker跑，
所有我把它移走了。

参照上面的**etcd.yaml**文件更改etcd配置文件 **/etc/default/etcd** 
```conf
## etcd(1) daemon options
## See "/usr/share/doc/etcd-server/op-guide/configuration.md.gz"

### Member flags

##### --name
## Human-readable name for this member.
## This value is referenced as this node's own entries listed in the
## `--initial-cluster` flag (e.g., `default=http://localhost:2380`). This
## needs to match the key used in the flag if using static bootstrapping. When
## using discovery, each member must have a unique name. `Hostname` or
## `machine-id` can be a good choice.
## default: "default"
ETCD_NAME="etcd-master1"

##### --data-dir
## Path to the data directory.
## default: "${name}.etcd"
#ETCD_DATA_DIR="/var/lib/etcd/master"

##### --wal-dir
## Path to the dedicated wal directory. If this flag is set, etcd will write
## the WAL files to the walDir rather than the dataDir. This allows a
## dedicated disk to be used, and helps avoid io competition between logging
## and other IO operations.
## default: ""
# ETCD_WAL_DIR

##### --snapshot-count
## Number of committed transactions to trigger a snapshot to disk.
## default: "100000"
# ETCD_SNAPSHOT_COUNT="100000"

##### --heartbeat-interval
## Time (in milliseconds) of a heartbeat interval.
## default: "100"
# ETCD_HEARTBEAT_INTERVAL="100"

##### --election-timeout
## Time (in milliseconds) for an election to timeout. See
## /usr/share/doc/etcd-server/tuning.md.gz for details.
## default: "1000"
# ETCD_ELECTION_TIMEOUT="1000"

##### --listen-peer-urls
## List of URLs to listen on for peer traffic. This flag tells the etcd to
## accept incoming requests from its peers on the specified scheme://IP:port
## combinations. Scheme can be either http or https.If 0.0.0.0 is specified as
## the IP, etcd listens to the given port on all interfaces. If an IP address is
## given as well as a port, etcd will listen on the given port and interface.
## Multiple URLs may be used to specify a number of addresses and ports to listen
## on. The etcd will respond to requests from any of the listed addresses and
## ports.
## default: "http://localhost:2380"
## example: "http://10.0.0.1:2380"
## invalid example: "http://example.com:2380" (domain name is invalid for binding)
# ETCD_LISTEN_PEER_URLS="http://localhost:2380"
ETCD_LISTEN_PEER_URLS="https://192.168.192.231:2380"

##### --listen-client-urls
## List of URLs to listen on for client traffic. This flag tells the etcd to
## accept incoming requests from the clients on the specified scheme://IP:port
## combinations. Scheme can be either http or https. If 0.0.0.0 is specified as
## the IP, etcd listens to the given port on all interfaces. If an IP address is
## given as well as a port, etcd will listen on the given port and interface.
## Multiple URLs may be used to specify a number of addresses and ports to listen
## on. The etcd will respond to requests from any of the listed addresses and
## ports.
## default: "http://localhost:2379"
## example: "http://10.0.0.1:2379"
## invalid example: "http://example.com:2379" (domain name is invalid for binding)
# ETCD_LISTEN_CLIENT_URLS="http://localhost:2379"
ETCD_LISTEN_CLIENT_URLS="https://192.168.192.231:2379"

##### --max-snapshots
## Maximum number of snapshot files to retain (0 is unlimited)
## The default for users on Windows is unlimited, and manual purging down to 5
## (or some preference for safety) is recommended.
## default: 5
# ETCD_MAX_SNAPSHOTS="5"

##### --max-wals
## Maximum number of wal files to retain (0 is unlimited)
## The default for users on Windows is unlimited, and manual purging down to 5
## (or some preference for safety) is recommended.
## default: 5
# ETCD_MAX_WALS="5"

##### --cors
## Comma-separated white list of origins for CORS (cross-origin resource
## sharing).
## default: none
# ETCD_CORS

### Clustering flags

# `--initial` prefix flags are used in bootstrapping (static bootstrap,
# discovery-service bootstrap or runtime reconfiguration) a new member, and
# ignored when restarting an existing member.

# `--discovery` prefix flags need to be set when using discovery service.

##### --initial-advertise-peer-urls

## List of this member's peer URLs to advertise to the rest of the cluster.
## These addresses are used for communicating etcd data around the cluster. At
## least one must be routable to all cluster members. These URLs can contain
## domain names.
## default: "http://localhost:2380"
## example: "http://example.com:2380, http://10.0.0.1:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://master1:2380"

##### --initial-cluster
## Initial cluster configuration for bootstrapping.
## The key is the value of the `--name` flag for each node provided. The
## default uses `default` for the key because this is the default for the
## `--name` flag.
## default: "default=http://localhost:2380"
ETCD_INITIAL_CLUSTER="etcd-master1=https://master1:2380,etcd-master2=https://master2:2380,etcd-master3=https://master3:2380"

##### --initial-cluster-state
## Initial cluster state ("new" or "existing"). Set to `new` for all members
## present during initial static or DNS bootstrapping. If this option is set to
## `existing`, etcd will attempt to join the existing cluster. If the wrong value
## is set, etcd will attempt to start but fail safely.
## default: "new"
ETCD_INITIAL_CLUSTER_STATE="new"

##### --initial-cluster-token
## Initial cluster token for the etcd cluster during bootstrap.
## default: "etcd-cluster"
# ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"

##### --advertise-client-urls
## List of this member's client URLs to advertise to the rest of the cluster.
## These URLs can contain domain names.
## Be careful if advertising URLs such as http://localhost:2379 from a cluster
## member and are using the proxy feature of etcd. This will cause loops, because
## the proxy will be forwarding requests to itself until its resources (memory,
## file descriptors) are eventually depleted.
## default: "http://localhost:2379"
## example: "http://example.com:2379, http://10.0.0.1:2379"
ETCD_ADVERTISE_CLIENT_URLS="https://master1:2379"

##### --discovery
## Discovery URL used to bootstrap the cluster.
## default: none
# ETCD_DISCOVERY

##### --discovery-srv
## DNS srv domain used to bootstrap the cluster.
## default: none
# ETCD_DISCOVERY_SRV

##### --discovery-fallback
## Expected behavior ("exit" or "proxy") when discovery services fails. "proxy"
## supports v2 API only.
## default: "proxy"
# ETCD_DISCOVERY_FALLBACK="proxy"

##### --discovery-proxy
## HTTP proxy to use for traffic to discovery service.
## default: none
# ETCD_DISCOVERY_PROXY

##### --strict-reconfig-check
## Reject reconfiguration requests that would cause quorum loss.
## default: false
# ETCD_STRICT_RECONFIG_CHECK

##### --auto-compaction-retention
## Auto compaction retention for mvcc key value store in hour. 0 means disable
## auto compaction.
## default: 0
# ETCD_AUTO_COMPACTION_RETENTION="0"


##### --enable-v2
## Accept etcd V2 client requests
## default: true
# ETCD_ENABLE_V2="true"

### Proxy flags

# `--proxy` prefix flags configures etcd to run in proxy mode. "proxy" supports
# v2 API only.

##### --proxy
## Proxy mode setting ("off", "readonly" or "on").
## default: "off"
# ETCD_PROXY="off"

##### --proxy-failure-wait
## Time (in milliseconds) an endpoint will be held in a failed state before
## being reconsidered for proxied requests.
## default: 5000
# ETCD_PROXY_FAILURE_WAIT="5000"

##### --proxy-refresh-interval
## Time (in milliseconds) of the endpoints refresh interval.
## default: 30000
# ETCD_PROXY_REFRESH_INTERVAL="30000"

##### --proxy-dial-timeout
## Time (in milliseconds) for a dial to timeout or 0 to disable the timeout
## default: 1000
# ETCD_PROXY_DIAL_TIMEOUT="1000"

##### --proxy-write-timeout
## Time (in milliseconds) for a write to timeout or 0 to disable the timeout.
## default: 5000
# ETCD_PROXY_WRITE_TIMEOUT="5000"

##### --proxy-read-timeout
## Time (in milliseconds) for a read to timeout or 0 to disable the timeout.
## Don't change this value if using watches because use long polling requests.
## default: 0
# ETCD_PROXY_READ_TIMEOUT="0"

### Security flags

# The security flags help to build a secure etcd cluster.

##### --ca-file (**DEPRECATED**)
## Path to the client server TLS CA file. `--ca-file ca.crt` could be replaced
## by `--trusted-ca-file ca.crt --client-cert-auth` and etcd will perform the
## same.
## default: none
# ETCD_CA_FILE

##### --cert-file
## Path to the client server TLS cert file.
## default: none
ETCD_CERT_FILE="/etc/kubernetes/pki/etcd/server.crt"

##### --key-file
## Path to the client server TLS key file.
## default: none
ETCD_KEY_FILE="/etc/kubernetes/pki/etcd/server.key"

##### --client-cert-auth
## Enable client cert authentication.
## default: false
ETCD_CLIENT_CERT_AUTH=true

##### --trusted-ca-file
## Path to the client server TLS trusted CA key file.
## default: none
ETCD_TRUSTED_CA_FILE="/etc/kubernetes/pki/etcd/ca.crt"

##### --auto-tls
## Client TLS using generated certificates
## default: false
ETCD_AUTO_TLS=true

##### --peer-ca-file (**DEPRECATED**)
## Path to the peer server TLS CA file. `--peer-ca-file ca.crt` could be
## replaced by `--peer-trusted-ca-file ca.crt --peer-client-cert-auth` and etcd
## will perform the same.
## default: none
ETCD_PEER_CA_FILE="/etc/kubernetes/pki/etcd/ca.crt"

##### --peer-cert-file
## Path to the peer server TLS cert file.
## default: none
ETCD_PEER_CERT_FILE="/etc/kubernetes/pki/etcd/peer.crt"

##### --peer-key-file
## Path to the peer server TLS key file.
## default: none
ETCD_PEER_KEY_FILE="/etc/kubernetes/pki/etcd/peer.key"

##### --peer-client-cert-auth
## Enable peer client cert authentication.
## default: false
ETCD_PEER_CLIENT_CERT_AUTH=true

##### --peer-trusted-ca-file
## Path to the peer server TLS trusted CA file.
## default: none
ETCD_PEER_TRUSTED_CA_FILE="/etc/kubernetes/pki/etcd/ca.crt"

##### --peer-auto-tls
## Peer TLS using generated certificates
## default: false
# ETCD_PEER_AUTO_TLS

### Logging flags

##### --debug
## Drop the default log level to DEBUG for all subpackages.
## default: false (INFO for all packages)
# ETCD_DEBUG

##### --log-package-levels
## Set individual etcd subpackages to specific log levels. An example being
## `etcdserver=WARNING,security=DEBUG`
## default: none (INFO for all packages)
# ETCD_LOG_PACKAGE_LEVELS


### Unsafe flags

# Please be CAUTIOUS when using unsafe flags because it will break the guarantees given by the consensus protocol.
# For example, it may panic if other members in the cluster are still alive.
# Follow the instructions when using these flags.

##### --force-new-cluster
## Force to create a new one-member cluster. It commits configuration changes
## forcing to remove all existing members in the cluster and add itself. It needs
## to be set to restore a backup.
## default: false
# ETCD_FORCE_NEW_CLUSTER
```
三台都改后重启etcd，etcd2:
```shell
# etcd 需要权限
sudo chmod 777 -R /etc/kubernetes/pik/etcd
sudo systemctl restart  etcd
# 查看状态
root@master1:~# etcdctl --endpoint=https://master2:2379 --cert-file=/etc/kubernetes/pki/etcd/server.crt  --key-file=/etc/kubernetes/pki/etcd/server.key --ca-file=/etc/kubernetes/pki/etcd/ca.crt cluster-health
member 5f23d73292959784 is healthy: got healthy result from https://master3:2379
member 9d8e4977362dd913 is healthy: got healthy result from https://master2:2379
member 9db5890b6392f376 is healthy: got healthy result from https://master1:2379
```
好了 etcd搭建好了。

## 初始化kubenetes集群


创建 初始化的配置文件
```shell
cat << EOF > init.yaml
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: stable
controlPlaneEndpoint: "192.168.255.254:16443"
etcd:
    external:
        endpoints:
        - https://192.168.192.231:3379
        - https://192.168.238.255:3379
        - https://192.168.238.167:3379
        caFile: /etc/kubernetes/pki/etcd/ca.crt
        certFile: /etc/kubernetes/pki/apiserver-etcd-client.crt
        keyFile: /etc/kubernetes/pki/apiserver-etcd-client.key
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/16
  podSubnet: 10.100.0.1/16
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
EOF
```
> - 上面配置设置使用了阿里云的镜像，因为 *k8s.gcr.io* 被墙了
> - 配置设置了 serviceSubnet podSubnet 安装网络插件这必须要设置的。

执行命令：
```shell
kubeadm init --config init.conf --upload-certs
```
当看到类似下面输出，说明你成功了。
```
You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join 192.168.255.254:16443 --token 74neap.19ou4fcojxvw4dow \
    --discovery-token-ca-cert-hash sha256:20f5a97e506b2a77e98622646490a6a6638e2ae4792bbe51bfd7c2c7ceb3196c \
    --control-plane --certificate-key cce07ad77f7dd5df1a70a190acc91cc8153bad24eeb52aa9140aa3fee2c38b13

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.255.254:16443 --token 74neap.19ou4fcojxvw4dow \
    --discovery-token-ca-cert-hash sha256:20f5a97e506b2a77e98622646490a6a6638e2ae4792bbe51bfd7c2c7ceb3196c
```

然后在其他两台都执行： 
```shell
kubeadm join 192.168.255.254:16443 --token 74neap.19ou4fcojxvw4dow \
    --discovery-token-ca-cert-hash sha256:20f5a97e506b2a77e98622646490a6a6638e2ae4792bbe51bfd7c2c7ceb3196c \
    --control-plane --certificate-key cce07ad77f7dd5df1a70a190acc91cc8153bad24eeb52aa9140aa3fee2c38b13

mkdir ~/.kube
cp /etc/kubernetes/admin.conf  ~/.kube/config
```
成功后在任意一台上可以看到下面的结果
```
root@master2:~# kubectl get node
NAME      STATUS     ROLES    AGE   VERSION
master1   NotReady   master   44s   v1.18.2
master2   NotReady   master   79s   v1.18.2
master3   NotReady   master   37s   v1.18.2
```

> status 是notready, 因为没有装网络插件

## 集群安装 **Calico**

在master上执行 
```
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```
等待确定所有的pod都是running状态
```sh
root@master1:~# kubectl get pods --all-namespaces
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-789f6df884-vpnjt   1/1     Running   0          22m
kube-system   calico-node-hxjdl                          1/1     Running   0          22m
kube-system   calico-node-jbfsl                          1/1     Running   0          22m
kube-system   calico-node-jqgrm                          1/1     Running   0          22m
kube-system   coredns-66bff467f8-kbw55                   1/1     Running   0          23m
kube-system   coredns-66bff467f8-pcrdj                   1/1     Running   0          23m
kube-system   kube-apiserver-master1                     1/1     Running   0          23m
kube-system   kube-apiserver-master2                     1/1     Running   0          23m
kube-system   kube-apiserver-master3                     1/1     Running   0          22m
kube-system   kube-controller-manager-master1            1/1     Running   0          23m
kube-system   kube-controller-manager-master2            1/1     Running   0          23m
kube-system   kube-controller-manager-master3            1/1     Running   0          22m
kube-system   kube-proxy-7nhnm                           1/1     Running   0          23m
kube-system   kube-proxy-fhxxk                           1/1     Running   0          23m
kube-system   kube-proxy-mq4pv                           1/1     Running   0          23m
kube-system   kube-scheduler-master1                     1/1     Running   0          23m
kube-system   kube-scheduler-master2                     1/1     Running   0          23m
kube-system   kube-scheduler-master3                     1/1     Running   0          21m
```
然后 
```shell
root@master2:~# kubectl taint nodes --all node-role.kubernetes.io/master-
node/master1 untainted
node/master2 untainted
node/master3 untainted
```

在看看node状态 ,STATUS 都是ready了
```shell
root@master1:~# kubectl get nodes -o wide
NAME      STATUS   ROLES    AGE   VERSION   INTERNAL-IP       EXTERNAL-IP   OS-IMAGE           KERNEL-VERSION     CONTAINER-RUNTIME
master1   Ready    master   25m   v1.18.2   192.168.192.231   <none>        Ubuntu 20.04 LTS   5.4.0-26-generic   docker://19.3.8
master2   Ready    master   26m   v1.18.2   192.168.238.255   <none>        Ubuntu 20.04 LTS   5.4.0-26-generic   docker://19.3.8
master3   Ready    master   25m   v1.18.2   192.168.238.167   <none>        Ubuntu 20.04 LTS   5.4.0-26-generic   docker://19.3.8
```

**后面就是随意添加node了**