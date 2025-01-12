---
title: "kubernets 里配置 Traefik"
date: 2020-05-20T09:22:19+08:00
draft: false
tags: ["kubernetes","how-to"]
categories: ["kubernetes"]
---

![traefik](/kubernetes/traefik-architecture.png)

> Træfɪk 是一个为了让部署微服务更加便捷而诞生的现代HTTP反向代理、负载均衡工具。 它支持多种后台 (Docker, Swarm, Kubernetes, Marathon, Mesos, Consul, Etcd, Zookeeper, BoltDB, Rest API, file…) 来自动化、动态的应用它的配置文件设置。

> traefik 通过Ingress规范来管理对群集服务的访问。

## 在 **kubernetes** 里配置 **Traefik**

### 定义 Traefik的 Kubernetes kind

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: ingressroutes.traefik.containo.us

spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: IngressRoute
    plural: ingressroutes
    singular: ingressroute
  scope: Namespaced

---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: middlewares.traefik.containo.us

spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: Middleware
    plural: middlewares
    singular: middleware
  scope: Namespaced

---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: ingressroutetcps.traefik.containo.us

spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: IngressRouteTCP
    plural: ingressroutetcps
    singular: ingressroutetcp
  scope: Namespaced

---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: ingressrouteudps.traefik.containo.us

spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: IngressRouteUDP
    plural: ingressrouteudps
    singular: ingressrouteudp
  scope: Namespaced

---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: tlsoptions.traefik.containo.us

spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: TLSOption
    plural: tlsoptions
    singular: tlsoption
  scope: Namespaced

---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: tlsstores.traefik.containo.us

spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: TLSStore
    plural: tlsstores
    singular: tlsstore
  scope: Namespaced

---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: traefikservices.traefik.containo.us

spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: TraefikService
    plural: traefikservices
    singular: traefikservice
  scope: Namespaced
```

> 上面定了：
> - IngressRoute：  路由，所有的入口
> -  Middleware： 中间件，如登陆等
> - IngressRouteTCP： TCP路由，如mysql的路由
> - IngressRouteUDP： UDP 路由
> - TLSOption： TLS 选项
> - TLSStore： TLS存储
> - TraefikService： traefik service



### 创建 RBAC
```yaml
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller

rules:
  - apiGroups:
      - ""
    resources:
      - services
      - endpoints
      - secrets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses/status
    verbs:
      - update
  - apiGroups:
      - traefik.containo.us
    resources:
      - middlewares
      - ingressroutes
      - traefikservices
      - ingressroutetcps
      - ingressrouteudps
      - tlsoptions
      - tlsstores
    verbs:
      - get
      - list
      - watch

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller

roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-ingress-controller
subjects:
  - kind: ServiceAccount
    name: traefik-ingress-controller
    namespace: default
```


### 部署 **Traefik**
- yaml示例：
  
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: default
  name: traefik-ingress-controller

---

apiVersion: v1
kind: ConfigMap
metadata:
  name: traefik
  namespace: default
data:
  traefik.yaml: |
    api:
      insecure: true
    log: {}
    entrypoints:
      web:
        address: :80
      websecure:
        address: :443
      websecureudp:
        address: :443/udp
    providers:
      kubernetescrd: true
    serverstransport:
      insecureskipverify: true
    certificatesresolvers:
      default:
        acme:
          httpChallenge:
            entryPoint: web
          email: goodking_bq@hotmail.com
          storage: /mnt/acme.json
    metrics:
      prometheus:
        entryPoint: web
---
kind: DaemonSet
apiVersion: apps/v1
metadata:
  namespace: default
  name: traefik
  labels:
    app: traefik

spec:
  selector:
    matchLabels:
      app: traefik
  template:
    metadata:
      labels:
        app: traefik
    spec:
      serviceAccountName: traefik-ingress-controller
      hostNetwork: true
      terminationGracePeriodSeconds: 60
      nodeSelector:             ## 设置node筛选器，在特定label的节点上启动
        IngressProxy: "true"
      containers:
        - name: traefik
          image: traefik:v2.2
          ports:
            - name: web
              containerPort: 80
              hostPort: 80
            - name: websecure
              containerPort: 443
              hostPort: 443
          securityContext:
            capabilities:
              drop:
                - ALL
              add:
                - NET_BIND_SERVICE
          volumeMounts:
            - name: nfs-pvc
              mountPath: "/mnt"
            - name: config
              mountPath: "/etc/traefik"
      volumes:
        - name: nfs-pvc
          persistentVolumeClaim:
            claimName: traefik
        - name: config
          configMap:
            name: traefik
```
- 说明
  
使用了configmap配置traefik

1. 我这里添加了tls支持，就是https(443)的支持,通过 `--entrypoints.websecure.Address=:443` 参数启用
2. 80 和 443 都用了 **hostPort** ,会在node上起相应的端口，通过端口可以访问服务
3. **Traefik**支持acme生成 *Let's Encrypt* 证书, 获取证书有三种模式,通过 `--certificatesresolvers.{name}.acme.{type}`配置

    - tlsChallenge: 
        > - 通过端口443验证证书
        ```yaml
        certificatesResolvers:
            myresolver:
                acme:
                # ...
                    tlsChallenge: {}
        ```
    - httpChallenge: ，
        > - 通过端口80验证证书
        > - `--certificatesresolvers.myresolver.acme.httpchallenge.entrypoint=80` 必须设置
        ```yaml
        entryPoints:
            web:
                address: ":80"

            websecure:
                address: ":443"

        certificatesResolvers:
            myresolver:
                acme:
                    # ...
                    httpChallenge:
                        entryPoint: web
        ```
    - dnsChallenge:
        > - 通过调域名提供商api添加一个域名text，验证。这需要额外的api认证参数
        > - 它支持很多域名商，具体可以看 [providers 列表](https://docs.traefik.io/https/acme/#providers)
        > - 它需要的参数都通过环境变量提供
        ```yml
        certificatesResolvers:
            myresolver:
                acme:
                    dnsChallenge:
                        provider: alidns
                        delayBeforeCheck: 0
        ```
4. **acme.json** 保存acme信息，避免重复申请证书，导致申请次数过多而失败

5. 取消了**traefik**和后端通信的tls验证，不需要了，因为**traefik**就支持tls, **--serverstransport.insecureskipverify** 参数实现
6. 这里使用**DaemonSets**部署而不是**Deployments**,每台node和master都有一个
    > - Deployments具有更轻松的上下扩展可能性。它可以实现完整的Pod生命周期并支持Kubernetes 1.2的滚动更新。至少需要一个Pod才能运行部署。
    > - DaemonSet会自动缩放到满足特定选择器的所有节点，并保证一次填充一个节点。Kubernetes 1.7还为DaemonSets全面支持滚动更新。
## 配置使用
### traefik 管理页面
  
```yaml 
# 创建service，配置端口
apiVersion: v1
kind: Service
metadata:
  name: traefik

spec:
  ports:
    - protocol: TCP
      name: web
      port: 80
    - protocol: TCP
      name: admin
      port: 8080
    - protocol: TCP
      name: websecure
      port: 443
  selector:
    app: traefik

--- 
# 创建IngressRoute，使用 traefik
# 同时使用了http和https,按需要添加
# tls.certResolver =default 是 --certificatesresolvers.default(name).acme.tlschallenge 配置的名字

apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: simpleingressroute-traefik
  namespace: default
spec:
  entryPoints:
    - web
  routes:
  - match: Host(`traefik.kube.example.com`)
    kind: Rule
    services:
    - name: traefik
      port: 8080

---

apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: ingressroutetls-traefik
  namespace: default
spec:
  entryPoints:
    - websecure
  routes:
  - match: Host(`traefik.kube.example.com`)
    kind: Rule
    services:
    - name: traefik
      port: 8080
  tls:
    certResolver: default
```

### kubernetes dashboard

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: ingressroutetls-dashboard
  namespace: default
spec:
  entryPoints:
    - websecure
  routes:
  - match: Host(`dashboard.kube.example.com`)
    kind: Rule
    services:
    - name: kubernetes-dashboard
      port: 443
  tls:
    certResolver: default
```

### mongodb 开放访问

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRouteTCP
metadata:
  name: mongodb
  namespace: yunwei
spec:
  entryPoints:
    - tcpep
  routes:
  - match: HostSNI(`mongo.kube.2xi.com`)
    services:
    - name: mongodb
      port: 27017
  tls:
    certResolver: default
```

> TCP UDP 使用的是**HostSNI**
> 
> HostSNI 如果要用域名，那么必须配置tls，我这里配置的是acm。
> 
> HostSNI 如果不使用tls，那么用 **\*** 代替域名,这个时候一个端口只能代理一个服务。

### 配置 **BasicAuth** 或是 **DigestAuth** 密码验证

> DigestAuth，和 BasicAuth 中间件是一种限制访问权限的简单方法。
> 
> DigestAuth 相比 BasicAuth ,更加安全
>
> DigestAuth 只支持realm = traefik

1. 生成密码
   > **BasicAuth** 和 **DigestAuth** 只是在生成密码的时候有区别，使用的时候都一样

   BasicAuth - 使用 **htpasswd** 命令生成

   DigestAuth - 使用 **htdigest** 命令生成
   ```shell
   # ubuntu 需要先安装以上两个命令
   sudo apt-get install apache2-utils
   # 生成 BasicAuth
   ➜  traefik htpasswd -cbB traefik admin admin
    Adding password for user admin
   ➜  traefik cat traefik
    admin:$2y$05$u81X.TPakpiyj1aexy9YZe3s58/.z91nY6D41ob4eYe8sW1uUZxOq
   # 生成 DigestAuth
   ➜  htdigest -c traefik traefik admin
    Adding password for admin in realm traefik.
    New password:
    Re-type new password:
   ➜  traefik cat traefik
    admin:traefik:234480a045a95d420967533e3e017b22
   ```

   
2. 使用 **kubectl** 创建 **secret**
   ```shell
   ➜  traefik kubectl create secret generic traefik-user  --from-file=users=traefik
    secret/traefik-user created
   kubectl describe secret traefik-user
   ```
3. 创建 middleware
   ```yaml
   apiVersion: traefik.containo.us/v1alpha1
   kind: Middleware
   metadata:
        name: traefik-digestauth
   spec:
        # basicAuth
        basicAuth:
            secret: traefik-user
        # 或是 digestAuth
        digestAuth:
            secret: traefik-user
   ```
4. 使用 middleware
   在 router 里添加使用 middleware
    ```yaml
    apiVersion: traefik.containo.us/v1alpha1
    kind: IngressRoute
    metadata:
    name: ingressroutetls-traefik
    namespace: default
    spec:
      entryPoints:
        - websecure
    routes:
      - match: Host(`traefik.kube.example.com`)
        kind: Rule
        services:
          - name: traefik
            port: 8080
        middlewares:
          - name: traefik-digestauth
    tls:
        certResolver: default
    ```

## 最后附上traefik命令帮助
```
root@192:~# docker run -it traefik:v2.2 traefik --help
traefik    Traefik is a modern HTTP reverse proxy and load balancer made to deploy microservices with ease.
Complete documentation is available at https://traefik.io

Usage: traefik [command] [flags] [arguments]

Use "traefik [command] --help" for help on any command.

Commands:
    healthcheck    Calls Traefik /ping endpoint (disabled by default) to check the health of Traefik.
    version        Shows the current Traefik version.

Flag's usage: traefik [--flag=flag_argument] [-f [flag_argument]]    # set flag_argument to flag(s)
          or: traefik [--flag[=true|false| ]] [-f [true|false| ]]    # set true/false to boolean flag(s)

Flags:
    --accesslog  (Default: "false")
        Access log settings.

    --accesslog.bufferingsize  (Default: "0")
        Number of access log lines to process in a buffered way.

    --accesslog.fields.defaultmode  (Default: "keep")
        Default mode for fields: keep | drop

    --accesslog.fields.headers.defaultmode  (Default: "drop")
        Default mode for fields: keep | drop | redact

    --accesslog.fields.headers.names.<name>  (Default: "")
        Override mode for headers

    --accesslog.fields.names.<name>  (Default: "")
        Override mode for fields

    --accesslog.filepath  (Default: "")
        Access log file path. Stdout is used when omitted or empty.

    --accesslog.filters.minduration  (Default: "0")
        Keep access logs when request took longer than the specified duration.

    --accesslog.filters.retryattempts  (Default: "false")
        Keep access logs when at least one retry happened.

    --accesslog.filters.statuscodes  (Default: "")
        Keep access logs with status codes in the specified range.

    --accesslog.format  (Default: "common")
        Access log format: json | common

    --api  (Default: "false")
        Enable api/dashboard.

    --api.dashboard  (Default: "true")
        Activate dashboard.

    --api.debug  (Default: "false")
        Enable additional endpoints for debugging and profiling.

    --api.insecure  (Default: "false")
        Activate API directly on the entryPoint named traefik.

    --certificatesresolvers.<name>  (Default: "false")
        Certificates resolvers configuration.

    --certificatesresolvers.<name>.acme.caserver  (Default: "https://acme-v02.api.letsencrypt.org/directory")
        CA server to use.

    --certificatesresolvers.<name>.acme.dnschallenge  (Default: "false")
        Activate DNS-01 Challenge.

    --certificatesresolvers.<name>.acme.dnschallenge.delaybeforecheck  (Default: "0")
        Assume DNS propagates after a delay in seconds rather than finding and querying
        nameservers.

    --certificatesresolvers.<name>.acme.dnschallenge.disablepropagationcheck  (Default: "false")
        Disable the DNS propagation checks before notifying ACME that the DNS challenge
        is ready. [not recommended]

    --certificatesresolvers.<name>.acme.dnschallenge.provider  (Default: "")
        Use a DNS-01 based challenge provider rather than HTTPS.

    --certificatesresolvers.<name>.acme.dnschallenge.resolvers  (Default: "")
        Use following DNS servers to resolve the FQDN authority.

    --certificatesresolvers.<name>.acme.email  (Default: "")
        Email address used for registration.

    --certificatesresolvers.<name>.acme.httpchallenge  (Default: "false")
        Activate HTTP-01 Challenge.

    --certificatesresolvers.<name>.acme.httpchallenge.entrypoint  (Default: "")
        HTTP challenge EntryPoint

    --certificatesresolvers.<name>.acme.keytype  (Default: "RSA4096")
        KeyType used for generating certificate private key. Allow value 'EC256',
        'EC384', 'RSA2048', 'RSA4096', 'RSA8192'.

    --certificatesresolvers.<name>.acme.storage  (Default: "acme.json")
        Storage to use.

    --certificatesresolvers.<name>.acme.tlschallenge  (Default: "true")
        Activate TLS-ALPN-01 Challenge.

    --configfile  (Default: "")
        Configuration file to use. If specified all other flags are ignored.

    --entrypoints.<name>  (Default: "false")
        Entry points definition.

    --entrypoints.<name>.address  (Default: "")
        Entry point address.

    --entrypoints.<name>.forwardedheaders.insecure  (Default: "false")
        Trust all forwarded headers.

    --entrypoints.<name>.forwardedheaders.trustedips  (Default: "")
        Trust only forwarded headers from selected IPs.

    --entrypoints.<name>.http  (Default: "")
        HTTP configuration.

    --entrypoints.<name>.http.middlewares  (Default: "")
        Default middlewares for the routers linked to the entry point.

    --entrypoints.<name>.http.redirections.entrypoint.permanent  (Default: "true")
        Applies a permanent redirection.

    --entrypoints.<name>.http.redirections.entrypoint.priority  (Default: "2147483647")
        Priority of the generated router.

    --entrypoints.<name>.http.redirections.entrypoint.scheme  (Default: "https")
        Scheme used for the redirection.

    --entrypoints.<name>.http.redirections.entrypoint.to  (Default: "")
        Targeted entry point of the redirection.

    --entrypoints.<name>.http.tls  (Default: "false")
        Default TLS configuration for the routers linked to the entry point.

    --entrypoints.<name>.http.tls.certresolver  (Default: "")
        Default certificate resolver for the routers linked to the entry point.

    --entrypoints.<name>.http.tls.domains  (Default: "")
        Default TLS domains for the routers linked to the entry point.

    --entrypoints.<name>.http.tls.domains[n].main  (Default: "")
        Default subject name.

    --entrypoints.<name>.http.tls.domains[n].sans  (Default: "")
        Subject alternative names.

    --entrypoints.<name>.http.tls.options  (Default: "")
        Default TLS options for the routers linked to the entry point.

    --entrypoints.<name>.proxyprotocol  (Default: "false")
        Proxy-Protocol configuration.

    --entrypoints.<name>.proxyprotocol.insecure  (Default: "false")
        Trust all.

    --entrypoints.<name>.proxyprotocol.trustedips  (Default: "")
        Trust only selected IPs.

    --entrypoints.<name>.transport.lifecycle.gracetimeout  (Default: "10")
        Duration to give active requests a chance to finish before Traefik stops.

    --entrypoints.<name>.transport.lifecycle.requestacceptgracetimeout  (Default: "0")
        Duration to keep accepting requests before Traefik initiates the graceful
        shutdown procedure.

    --entrypoints.<name>.transport.respondingtimeouts.idletimeout  (Default: "180")
        IdleTimeout is the maximum amount duration an idle (keep-alive) connection will
        remain idle before closing itself. If zero, no timeout is set.

    --entrypoints.<name>.transport.respondingtimeouts.readtimeout  (Default: "0")
        ReadTimeout is the maximum duration for reading the entire request, including
        the body. If zero, no timeout is set.

    --entrypoints.<name>.transport.respondingtimeouts.writetimeout  (Default: "0")
        WriteTimeout is the maximum duration before timing out writes of the response.
        If zero, no timeout is set.

    --global.checknewversion  (Default: "true")
        Periodically check if a new version has been released.

    --global.sendanonymoususage
        Periodically send anonymous usage statistics. If the option is not specified, it
        will be enabled by default.

    --hostresolver  (Default: "false")
        Enable CNAME Flattening.

    --hostresolver.cnameflattening  (Default: "false")
        A flag to enable/disable CNAME flattening

    --hostresolver.resolvconfig  (Default: "/etc/resolv.conf")
        resolv.conf used for DNS resolving

    --hostresolver.resolvdepth  (Default: "5")
        The maximal depth of DNS recursive resolving

    --log  (Default: "false")
        Traefik log settings.

    --log.filepath  (Default: "")
        Traefik log file path. Stdout is used when omitted or empty.

    --log.format  (Default: "common")
        Traefik log format: json | common

    --log.level  (Default: "ERROR")
        Log level set to traefik logs.

    --metrics.datadog  (Default: "false")
        Datadog metrics exporter type.

    --metrics.datadog.addentrypointslabels  (Default: "true")
        Enable metrics on entry points.

    --metrics.datadog.address  (Default: "localhost:8125")
        Datadog's address.

    --metrics.datadog.addserviceslabels  (Default: "true")
        Enable metrics on services.

    --metrics.datadog.pushinterval  (Default: "10")
        Datadog push interval.

    --metrics.influxdb  (Default: "false")
        InfluxDB metrics exporter type.

    --metrics.influxdb.addentrypointslabels  (Default: "true")
        Enable metrics on entry points.

    --metrics.influxdb.address  (Default: "localhost:8089")
        InfluxDB address.

    --metrics.influxdb.addserviceslabels  (Default: "true")
        Enable metrics on services.

    --metrics.influxdb.database  (Default: "")
        InfluxDB database used when protocol is http.

    --metrics.influxdb.password  (Default: "")
        InfluxDB password (only with http).

    --metrics.influxdb.protocol  (Default: "udp")
        InfluxDB address protocol (udp or http).

    --metrics.influxdb.pushinterval  (Default: "10")
        InfluxDB push interval.

    --metrics.influxdb.retentionpolicy  (Default: "")
        InfluxDB retention policy used when protocol is http.

    --metrics.influxdb.username  (Default: "")
        InfluxDB username (only with http).

    --metrics.prometheus  (Default: "false")
        Prometheus metrics exporter type.

    --metrics.prometheus.addentrypointslabels  (Default: "true")
        Enable metrics on entry points.

    --metrics.prometheus.addserviceslabels  (Default: "true")
        Enable metrics on services.

    --metrics.prometheus.buckets  (Default: "0.100000, 0.300000, 1.200000, 5.000000")
        Buckets for latency metrics.

    --metrics.prometheus.entrypoint  (Default: "traefik")
        EntryPoint

    --metrics.prometheus.manualrouting  (Default: "false")
        Manual routing

    --metrics.statsd  (Default: "false")
        StatsD metrics exporter type.

    --metrics.statsd.addentrypointslabels  (Default: "true")
        Enable metrics on entry points.

    --metrics.statsd.address  (Default: "localhost:8125")
        StatsD address.

    --metrics.statsd.addserviceslabels  (Default: "true")
        Enable metrics on services.

    --metrics.statsd.prefix  (Default: "traefik")
        Prefix to use for metrics collection.

    --metrics.statsd.pushinterval  (Default: "10")
        StatsD push interval.

    --ping  (Default: "false")
        Enable ping.

    --ping.entrypoint  (Default: "traefik")
        EntryPoint

    --ping.manualrouting  (Default: "false")
        Manual routing

    --providers.consul  (Default: "false")
        Enable Consul backend with default settings.

    --providers.consul.endpoints  (Default: "127.0.0.1:8500")
        KV store endpoints

    --providers.consul.password  (Default: "")
        KV Password

    --providers.consul.rootkey  (Default: "traefik")
        Root key used for KV store

    --providers.consul.tls.ca  (Default: "")
        TLS CA

    --providers.consul.tls.caoptional  (Default: "false")
        TLS CA.Optional

    --providers.consul.tls.cert  (Default: "")
        TLS cert

    --providers.consul.tls.insecureskipverify  (Default: "false")
        TLS insecure skip verify

    --providers.consul.tls.key  (Default: "")
        TLS key

    --providers.consul.username  (Default: "")
        KV Username

    --providers.consulcatalog.cache  (Default: "false")
        Use local agent caching for catalog reads.

    --providers.consulcatalog.constraints  (Default: "")
        Constraints is an expression that Traefik matches against the container's labels
        to determine whether to create any route for that container.

    --providers.consulcatalog.defaultrule  (Default: "Host(`{{ normalize .Name }}`)")
        Default rule.

    --providers.consulcatalog.endpoint.address  (Default: "http://127.0.0.1:8500")
        The address of the Consul server

    --providers.consulcatalog.endpoint.datacenter  (Default: "")
        Data center to use. If not provided, the default agent data center is used

    --providers.consulcatalog.endpoint.endpointwaittime  (Default: "0")
        WaitTime limits how long a Watch will block. If not provided, the agent default
        values will be used

    --providers.consulcatalog.endpoint.httpauth.password  (Default: "")
        Basic Auth password

    --providers.consulcatalog.endpoint.httpauth.username  (Default: "")
        Basic Auth username

    --providers.consulcatalog.endpoint.scheme  (Default: "")
        The URI scheme for the Consul server

    --providers.consulcatalog.endpoint.tls.ca  (Default: "")
        TLS CA

    --providers.consulcatalog.endpoint.tls.caoptional  (Default: "false")
        TLS CA.Optional

    --providers.consulcatalog.endpoint.tls.cert  (Default: "")
        TLS cert

    --providers.consulcatalog.endpoint.tls.insecureskipverify  (Default: "false")
        TLS insecure skip verify

    --providers.consulcatalog.endpoint.tls.key  (Default: "")
        TLS key

    --providers.consulcatalog.endpoint.token  (Default: "")
        Token is used to provide a per-request ACL token which overrides the agent's
        default token

    --providers.consulcatalog.exposedbydefault  (Default: "true")
        Expose containers by default.

    --providers.consulcatalog.prefix  (Default: "traefik")
        Prefix for consul service tags. Default 'traefik'

    --providers.consulcatalog.refreshinterval  (Default: "15")
        Interval for check Consul API. Default 100ms

    --providers.consulcatalog.requireconsistent  (Default: "false")
        Forces the read to be fully consistent.

    --providers.consulcatalog.stale  (Default: "false")
        Use stale consistency for catalog reads.

    --providers.docker  (Default: "false")
        Enable Docker backend with default settings.

    --providers.docker.constraints  (Default: "")
        Constraints is an expression that Traefik matches against the container's labels
        to determine whether to create any route for that container.

    --providers.docker.defaultrule  (Default: "Host(`{{ normalize .Name }}`)")
        Default rule.

    --providers.docker.endpoint  (Default: "unix:///var/run/docker.sock")
        Docker server endpoint. Can be a tcp or a unix socket endpoint.

    --providers.docker.exposedbydefault  (Default: "true")
        Expose containers by default.

    --providers.docker.network  (Default: "")
        Default Docker network used.

    --providers.docker.swarmmode  (Default: "false")
        Use Docker on Swarm Mode.

    --providers.docker.swarmmoderefreshseconds  (Default: "15")
        Polling interval for swarm mode.

    --providers.docker.tls.ca  (Default: "")
        TLS CA

    --providers.docker.tls.caoptional  (Default: "false")
        TLS CA.Optional

    --providers.docker.tls.cert  (Default: "")
        TLS cert

    --providers.docker.tls.insecureskipverify  (Default: "false")
        TLS insecure skip verify

    --providers.docker.tls.key  (Default: "")
        TLS key

    --providers.docker.usebindportip  (Default: "false")
        Use the ip address from the bound port, rather than from the inner network.

    --providers.docker.watch  (Default: "true")
        Watch Docker Swarm events.

    --providers.etcd  (Default: "false")
        Enable Etcd backend with default settings.

    --providers.etcd.endpoints  (Default: "127.0.0.1:2379")
        KV store endpoints

    --providers.etcd.password  (Default: "")
        KV Password

    --providers.etcd.rootkey  (Default: "traefik")
        Root key used for KV store

    --providers.etcd.tls.ca  (Default: "")
        TLS CA

    --providers.etcd.tls.caoptional  (Default: "false")
        TLS CA.Optional

    --providers.etcd.tls.cert  (Default: "")
        TLS cert

    --providers.etcd.tls.insecureskipverify  (Default: "false")
        TLS insecure skip verify

    --providers.etcd.tls.key  (Default: "")
        TLS key

    --providers.etcd.username  (Default: "")
        KV Username

    --providers.file.debugloggeneratedtemplate  (Default: "false")
        Enable debug logging of generated configuration template.

    --providers.file.directory  (Default: "")
        Load dynamic configuration from one or more .toml or .yml files in a directory.

    --providers.file.filename  (Default: "")
        Load dynamic configuration from a file.

    --providers.file.watch  (Default: "true")
        Watch provider.

    --providers.kubernetescrd  (Default: "false")
        Enable Kubernetes backend with default settings.

    --providers.kubernetescrd.certauthfilepath  (Default: "")
        Kubernetes certificate authority file path (not needed for in-cluster client).

    --providers.kubernetescrd.disablepasshostheaders  (Default: "false")
        Kubernetes disable PassHost Headers.

    --providers.kubernetescrd.endpoint  (Default: "")
        Kubernetes server endpoint (required for external cluster client).

    --providers.kubernetescrd.ingressclass  (Default: "")
        Value of kubernetes.io/ingress.class annotation to watch for.

    --providers.kubernetescrd.labelselector  (Default: "")
        Kubernetes label selector to use.

    --providers.kubernetescrd.namespaces  (Default: "")
        Kubernetes namespaces.

    --providers.kubernetescrd.throttleduration  (Default: "0")
        Ingress refresh throttle duration

    --providers.kubernetescrd.token  (Default: "")
        Kubernetes bearer token (not needed for in-cluster client).

    --providers.kubernetesingress  (Default: "false")
        Enable Kubernetes backend with default settings.

    --providers.kubernetesingress.certauthfilepath  (Default: "")
        Kubernetes certificate authority file path (not needed for in-cluster client).

    --providers.kubernetesingress.disablepasshostheaders  (Default: "false")
        Kubernetes disable PassHost Headers.

    --providers.kubernetesingress.endpoint  (Default: "")
        Kubernetes server endpoint (required for external cluster client).

    --providers.kubernetesingress.ingressclass  (Default: "")
        Value of kubernetes.io/ingress.class annotation to watch for.

    --providers.kubernetesingress.ingressendpoint.hostname  (Default: "")
        Hostname used for Kubernetes Ingress endpoints.

    --providers.kubernetesingress.ingressendpoint.ip  (Default: "")
        IP used for Kubernetes Ingress endpoints.

    --providers.kubernetesingress.ingressendpoint.publishedservice  (Default: "")
        Published Kubernetes Service to copy status from.

    --providers.kubernetesingress.labelselector  (Default: "")
        Kubernetes Ingress label selector to use.

    --providers.kubernetesingress.namespaces  (Default: "")
        Kubernetes namespaces.

    --providers.kubernetesingress.throttleduration  (Default: "0")
        Ingress refresh throttle duration

    --providers.kubernetesingress.token  (Default: "")
        Kubernetes bearer token (not needed for in-cluster client).

    --providers.marathon  (Default: "false")
        Enable Marathon backend with default settings.

    --providers.marathon.basic.httpbasicauthuser  (Default: "")
        Basic authentication User.

    --providers.marathon.basic.httpbasicpassword  (Default: "")
        Basic authentication Password.

    --providers.marathon.constraints  (Default: "")
        Constraints is an expression that Traefik matches against the application's
        labels to determine whether to create any route for that application.

    --providers.marathon.dcostoken  (Default: "")
        DCOSToken for DCOS environment, This will override the Authorization header.

    --providers.marathon.defaultrule  (Default: "Host(`{{ normalize .Name }}`)")
        Default rule.

    --providers.marathon.dialertimeout  (Default: "5")
        Set a dialer timeout for Marathon.

    --providers.marathon.endpoint  (Default: "http://127.0.0.1:8080")
        Marathon server endpoint. You can also specify multiple endpoint for Marathon.

    --providers.marathon.exposedbydefault  (Default: "true")
        Expose Marathon apps by default.

    --providers.marathon.forcetaskhostname  (Default: "false")
        Force to use the task's hostname.

    --providers.marathon.keepalive  (Default: "10")
        Set a TCP Keep Alive time.

    --providers.marathon.respectreadinesschecks  (Default: "false")
        Filter out tasks with non-successful readiness checks during deployments.

    --providers.marathon.responseheadertimeout  (Default: "60")
        Set a response header timeout for Marathon.

    --providers.marathon.tls.ca  (Default: "")
        TLS CA

    --providers.marathon.tls.caoptional  (Default: "false")
        TLS CA.Optional

    --providers.marathon.tls.cert  (Default: "")
        TLS cert

    --providers.marathon.tls.insecureskipverify  (Default: "false")
        TLS insecure skip verify

    --providers.marathon.tls.key  (Default: "")
        TLS key

    --providers.marathon.tlshandshaketimeout  (Default: "5")
        Set a TLS handshake timeout for Marathon.

    --providers.marathon.trace  (Default: "false")
        Display additional provider logs.

    --providers.marathon.watch  (Default: "true")
        Watch provider.

    --providers.providersthrottleduration  (Default: "2")
        Backends throttle duration: minimum duration between 2 events from providers
        before applying a new configuration. It avoids unnecessary reloads if multiples
        events are sent in a short amount of time.

    --providers.rancher  (Default: "false")
        Enable Rancher backend with default settings.

    --providers.rancher.constraints  (Default: "")
        Constraints is an expression that Traefik matches against the container's labels
        to determine whether to create any route for that container.

    --providers.rancher.defaultrule  (Default: "Host(`{{ normalize .Name }}`)")
        Default rule.

    --providers.rancher.enableservicehealthfilter  (Default: "true")
        Filter services with unhealthy states and inactive states.

    --providers.rancher.exposedbydefault  (Default: "true")
        Expose containers by default.

    --providers.rancher.intervalpoll  (Default: "false")
        Poll the Rancher metadata service every 'rancher.refreshseconds' (less
        accurate).

    --providers.rancher.prefix  (Default: "latest")
        Prefix used for accessing the Rancher metadata service.

    --providers.rancher.refreshseconds  (Default: "15")
        Defines the polling interval in seconds.

    --providers.rancher.watch  (Default: "true")
        Watch provider.

    --providers.redis  (Default: "false")
        Enable Redis backend with default settings.

    --providers.redis.endpoints  (Default: "127.0.0.1:6379")
        KV store endpoints

    --providers.redis.password  (Default: "")
        KV Password

    --providers.redis.rootkey  (Default: "traefik")
        Root key used for KV store

    --providers.redis.tls.ca  (Default: "")
        TLS CA

    --providers.redis.tls.caoptional  (Default: "false")
        TLS CA.Optional

    --providers.redis.tls.cert  (Default: "")
        TLS cert

    --providers.redis.tls.insecureskipverify  (Default: "false")
        TLS insecure skip verify

    --providers.redis.tls.key  (Default: "")
        TLS key

    --providers.redis.username  (Default: "")
        KV Username

    --providers.rest  (Default: "false")
        Enable Rest backend with default settings.

    --providers.rest.insecure  (Default: "false")
        Activate REST Provider directly on the entryPoint named traefik.

    --providers.zookeeper  (Default: "false")
        Enable ZooKeeper backend with default settings.

    --providers.zookeeper.endpoints  (Default: "127.0.0.1:2181")
        KV store endpoints

    --providers.zookeeper.password  (Default: "")
        KV Password

    --providers.zookeeper.rootkey  (Default: "traefik")
        Root key used for KV store

    --providers.zookeeper.tls.ca  (Default: "")
        TLS CA

    --providers.zookeeper.tls.caoptional  (Default: "false")
        TLS CA.Optional

    --providers.zookeeper.tls.cert  (Default: "")
        TLS cert

    --providers.zookeeper.tls.insecureskipverify  (Default: "false")
        TLS insecure skip verify

    --providers.zookeeper.tls.key  (Default: "")
        TLS key

    --providers.zookeeper.username  (Default: "")
        KV Username

    --serverstransport.forwardingtimeouts.dialtimeout  (Default: "30")
        The amount of time to wait until a connection to a backend server can be
        established. If zero, no timeout exists.

    --serverstransport.forwardingtimeouts.idleconntimeout  (Default: "90")
        The maximum period for which an idle HTTP keep-alive connection will remain open
        before closing itself

    --serverstransport.forwardingtimeouts.responseheadertimeout  (Default: "0")
        The amount of time to wait for a server's response headers after fully writing
        the request (including its body, if any). If zero, no timeout exists.

    --serverstransport.insecureskipverify  (Default: "false")
        Disable SSL certificate verification.

    --serverstransport.maxidleconnsperhost  (Default: "200")
        If non-zero, controls the maximum idle (keep-alive) to keep per-host. If zero,
        DefaultMaxIdleConnsPerHost is used

    --serverstransport.rootcas  (Default: "")
        Add cert file for self-signed certificate.

    --tracing  (Default: "false")
        OpenTracing configuration.

    --tracing.datadog  (Default: "false")
        Settings for Datadog.

    --tracing.datadog.bagageprefixheadername  (Default: "")
        Specifies the header name prefix that will be used to store baggage items in a
        map.

    --tracing.datadog.debug  (Default: "false")
        Enable Datadog debug.

    --tracing.datadog.globaltag  (Default: "")
        Key:Value tag to be set on all the spans.

    --tracing.datadog.localagenthostport  (Default: "localhost:8126")
        Set datadog-agent's host:port that the reporter will used.

    --tracing.datadog.parentidheadername  (Default: "")
        Specifies the header name that will be used to store the parent ID.

    --tracing.datadog.prioritysampling  (Default: "false")
        Enable priority sampling. When using distributed tracing, this option must be
        enabled in order to get all the parts of a distributed trace sampled.

    --tracing.datadog.samplingpriorityheadername  (Default: "")
        Specifies the header name that will be used to store the sampling priority.

    --tracing.datadog.traceidheadername  (Default: "")
        Specifies the header name that will be used to store the trace ID.

    --tracing.elastic  (Default: "false")
        Settings for Elastic.

    --tracing.elastic.secrettoken  (Default: "")
        Set the token used to connect to Elastic APM Server.

    --tracing.elastic.serverurl  (Default: "")
        Set the URL of the Elastic APM server.

    --tracing.elastic.serviceenvironment  (Default: "")
        Set the name of the environment Traefik is deployed in, e.g. 'production' or
        'staging'.

    --tracing.haystack  (Default: "false")
        Settings for Haystack.

    --tracing.haystack.baggageprefixheadername  (Default: "")
        Specifies the header name prefix that will be used to store baggage items in a
        map.

    --tracing.haystack.globaltag  (Default: "")
        Key:Value tag to be set on all the spans.

    --tracing.haystack.localagenthost  (Default: "127.0.0.1")
        Set haystack-agent's host that the reporter will used.

    --tracing.haystack.localagentport  (Default: "35000")
        Set haystack-agent's port that the reporter will used.

    --tracing.haystack.parentidheadername  (Default: "")
        Specifies the header name that will be used to store the parent ID.

    --tracing.haystack.spanidheadername  (Default: "")
        Specifies the header name that will be used to store the span ID.

    --tracing.haystack.traceidheadername  (Default: "")
        Specifies the header name that will be used to store the trace ID.

    --tracing.instana  (Default: "false")
        Settings for Instana.

    --tracing.instana.localagenthost  (Default: "")
        Set instana-agent's host that the reporter will used.

    --tracing.instana.localagentport  (Default: "42699")
        Set instana-agent's port that the reporter will used.

    --tracing.instana.loglevel  (Default: "info")
        Set instana-agent's log level. ('error','warn','info','debug')

    --tracing.jaeger  (Default: "false")
        Settings for Jaeger.

    --tracing.jaeger.collector.endpoint  (Default: "")
        Instructs reporter to send spans to jaeger-collector at this URL.

    --tracing.jaeger.collector.password  (Default: "")
        Password for basic http authentication when sending spans to jaeger-collector.

    --tracing.jaeger.collector.user  (Default: "")
        User for basic http authentication when sending spans to jaeger-collector.

    --tracing.jaeger.gen128bit  (Default: "false")
        Generate 128 bit span IDs.

    --tracing.jaeger.localagenthostport  (Default: "127.0.0.1:6831")
        Set jaeger-agent's host:port that the reporter will used.

    --tracing.jaeger.propagation  (Default: "jaeger")
        Which propagation format to use (jaeger/b3).

    --tracing.jaeger.samplingparam  (Default: "1.000000")
        Set the sampling parameter.

    --tracing.jaeger.samplingserverurl  (Default: "http://localhost:5778/sampling")
        Set the sampling server url.

    --tracing.jaeger.samplingtype  (Default: "const")
        Set the sampling type.

    --tracing.jaeger.tracecontextheadername  (Default: "uber-trace-id")
        Set the header to use for the trace-id.

    --tracing.servicename  (Default: "traefik")
        Set the name for this service.

    --tracing.spannamelimit  (Default: "0")
        Set the maximum character limit for Span names (default 0 = no limit).

    --tracing.zipkin  (Default: "false")
        Settings for Zipkin.

    --tracing.zipkin.httpendpoint  (Default: "http://localhost:9411/api/v2/spans")
        HTTP Endpoint to report traces to.

    --tracing.zipkin.id128bit  (Default: "true")
        Use Zipkin 128 bit root span IDs.

    --tracing.zipkin.samespan  (Default: "false")
        Use Zipkin SameSpan RPC style traces.

    --tracing.zipkin.samplerate  (Default: "1.000000")
        The rate between 0.0 and 1.0 of requests to trace.
```