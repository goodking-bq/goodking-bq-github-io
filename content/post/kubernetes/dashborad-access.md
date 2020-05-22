---
title: "Kubernetes Dashborad 配置访问"
date: 2020-05-22T10:43:43+08:00
draft: false
tags: ["kubernetes","how-to"]
categories: ["kubernetes"]
---

> Kubernetes Dashborad 有两种认证方式
> - TOKEN： 每个 Service Account 都有一个 valid Bearer Token ，可用于登录 Dashboard
> - Kubeconfig：使用创建的kubeconfig文件以配置对集群的访问权限来登陆

## **Token**
### 先创建 **User** ,并给 **User** 角色

Kubernetes 里面每个 ServiceAccount 都对应一个secret，而每个secret都对应了一个Token值，登陆需要的就是这个值
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard

---

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
```
创建了用户admin-user，并给了他cluster-admin角色，这个角色管理集群权限

### 查看 **Token**

上面创建的 sa:admin-user,会自动创建一个secret,
```sh
➜  ~ kubectl get sa,secret -n kubernetes-dashboard
NAME                                  SECRETS   AGE
serviceaccount/admin-user             1         25h
serviceaccount/default                1         25h
serviceaccount/kubernetes-dashboard   1         25h

NAME                                      TYPE                                  DATA   AGE
secret/admin-user-token-6fk7j             kubernetes.io/service-account-token   3      25h
secret/default-token-977cp                kubernetes.io/service-account-token   3      25h
secret/kubernetes-dashboard-certs         Opaque                                0      25h
secret/kubernetes-dashboard-csrf          Opaque                                1      25h
secret/kubernetes-dashboard-key-holder    Opaque                                2      25h
secret/kubernetes-dashboard-token-fc428   kubernetes.io/service-account-token   3      25h
```

只需要获取 **admin-user-token-6fk7j**  的token 就可以登录了。
```shell
➜  ~ kubectl -n kubernetes-dashboard describe secret admin-user-token-6fk7j
Name:         admin-user-token-6fk7j
Namespace:    kubernetes-dashboard
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: admin-user
              kubernetes.io/service-account.uid: bb0d606c-e382-4d73-869f-3cc9ced4f9d3

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  20 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6ImhSR2o0OUZMRTNfX3BILTF3X3FUR19Id3p5Y205am1OLVViNHhhUEFtS1kifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLTZmazdqIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJiYjBkNjA2Yy1lMzgyLTRkNzMtODY5Zi0zY2M5Y2VkNGY5ZDMiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZXJuZXRlcy1kYXNoYm9hcmQ6YWRtaW4tdXNlciJ9.***
```

## **Kubeconfig**

**kubeconfig** 在将集群、用户和上下文定义在一个或多个配置文件中之后，用户可以使用 `kubectl config use-context` 命令快速地在集群之间进行切换。

1. 使用token方式
```sh
# 获取token
➜  ~ TOKEN=$(kubectl -n kubernetes-dashboard get secret admin-user-token-6fk7j -o jsonpath={.data.token}|base64 -d)
# 设置cluster kube 以及他的 server
➜  ~ kubectl config --kubeconfig=dashboard-admin.conf set-cluster kube --server=https://192.168.255.254:16443
Cluster "kube" set.
# 设置用户的token
➜  ~ kubectl config --kubeconfig=dashboard-admin.conf set-credentials admin-user --token=$TOKEN
User "admin-user" set.
# 设置用户与集群的绑定 context
➜  ~ kubectl config --kubeconfig=dashboard-admin.conf set-context admin-user@kubernetes --cluster=kube --user=admin-user
Context "admin-user@kubernetes" created.
# 当前使用admin-user@kubernetes context.设置默认
➜  ~ kubectl config --kubeconfig=dashboard-admin.conf use-context admin-user@kubernetes
Switched to context "admin-user@kubernetes".
# 查看
➜  ~ cat dashboard-admin.conf
apiVersion: v1
clusters:
- cluster:
    server: https://192.168.255.254:16443
  name: kube
contexts:
- context:
    cluster: kube
    user: admin-user
  name: admin-user@kubernetes
current-context: admin-user@kubernetes
kind: Config
preferences: {}
users:
- name: admin-user
  user:
    token: eyJhbGciOiJSUzI1NiIsImtpZCI6ImhSR2o0OUZMRTNfX3BILTF3X3FUR19Id3p5Y205am1OLVViNHhhUEFtS1kifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLTZmazdqIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJiYjBkNjA2Yy1lMzgyLTRkNzMtODY5Zi0zY2M5Y2VkNGY5ZDMiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZXJuZXRlcy1kYXNoYm9hcmQ6YWRtaW4tdXNlciJ9.***
```

同理 你可以在加 **user** ,**cluster**, **context**,执行 **kubectl config use-context** 或是 **kubectl  get node --kubeconfig=./kubeconfig --context=cluster1-context** 快速在机器之间切换