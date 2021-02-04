---
title: "Install K8s on Single Cluser"
date: 2021-02-03T10:36:25+08:00
date: '2021-02-03'
lastModified: '2021-02-03'
description: ''
author: 'dashjay'
tags: ["docker", "kubernetes"]
slug: install-k8s-on-single-cluser
---


## 前言

虽然装环境学习本身是比较简单的，使用的都是第三方工具，然而因为环境因人而异，教程中英搬运混合复杂，导致我在这上面竟然浪费了几个小时，因此写个日记记录一下，也算是我开始了我的k8s之路。

## 环境准备

因为本次是单机安装，因此仅仅只准备了一台18.04 ubuntu-server 的机器

```bash
$ cat /proc/version
Linux version 4.15.0-135-generic (buildd@lgw01-amd64-005) \
 (gcc version 7.5.0 (Ubuntu 7.5.0-3ubuntu1~18.04)) \
 #139-Ubuntu SMP Mon Jan 18 17:38:24 UTC 2021
```

部署 k8s 的时候考虑到性能问题，安全性问题，可用性问题需要做以下操作：

- 关闭 swap
- 关闭 防火墙

执行以下命令即可

```bash
sudo swapoff -a
sudo ufw disable
```

## Step1 导入Key安装源

```cpp
apt udpate
apt-get -y install apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add -
```

sudo 编辑源

```cpp
$ cat /etc/apt/sources.list.d/kubernetes.list
deb http://mirrors.ustc.edu.cn/kubernetes/apt kubernetes-xenial main
$ apt update 
```

## Step2 安装启动 k8s

```cpp
$ sudo apt-get install -y kubelet kubeadm kubectl
```

安装完了检查版本

```cpp
$ kubelet --version
Kubernetes v1.20.2
```

初始化 k8s

```cpp
$ kubeadm init \
  --kubernetes-version=v1.20.2 \
  --image-repository registry.aliyuncs.com/google_containers \
  --pod-network-cidr=10.24.0.0/16 \
  --ignore-preflight-errors=Swap
```

要使用这个节点，你需要去执行几个命令：

```cpp
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

执行下列命令等待所有服务进入 Running 状态，并且数量满足预期

```cpp
kubectl get pods --all-namespaces
```

```cpp
// 输出
➜  ~ kubectl get pods --all-namespaces
NAMESPACE     NAME                                 READY   STATUS    RESTARTS   AGE
kube-system   coredns-7f89b7bc75-48dpk             1/1     Running   0          18m
kube-system   coredns-7f89b7bc75-zns9h             1/1     Running   0          18m
kube-system   etcd-k8s-master                      1/1     Running   0          18m
kube-system   kube-apiserver-k8s-master            1/1     Running   0          18m
kube-system   kube-controller-manager-k8s-master   1/1     Running   0          18m
kube-system   kube-proxy-4jjg7                     1/1     Running   0          18m
kube-system   kube-scheduler-k8s-master            1/1     Running   0          18m
```

如果 READY 中存在 0/1 或者 STATUS 不为 RUNNING 则可能存在其他情况，之后要进一步排除

## Step3 安装 DashBoard 可视化管理

其实上一步已经完成了安装了，不过一般情况下，教程里都会教你如何安装可视化管理后台Dashboard

下载推荐安装方式的yaml文件 `wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.1.0/aio/deploy/recommended.yaml`

下载之后 `kubectl apply -f recommended.yaml` 即可应用，如果github不方便访问可以点击此链接下载，文件不大，我已经上传到博客中

[recommended.yaml](/post/2021-02-03-install-k8s-on-single-cluser_files/recommended.yaml)

再次执行查看是否全部就绪

```cpp
kubectl get pods --all-namespaces
```

输出

```cpp
➜  ~ kubectl get pods --all-namespaces
NAMESPACE              NAME                                         READY   STATUS    RESTARTS   AGE
kube-system            coredns-7f89b7bc75-48dpk                     1/1     Running   0          34m
kube-system            coredns-7f89b7bc75-zns9h                     1/1     Running   0          34m
kube-system            etcd-k8s-master                              1/1     Running   0          34m
kube-system            kube-apiserver-k8s-master                    1/1     Running   0          34m
kube-system            kube-controller-manager-k8s-master           1/1     Running   0          34m
kube-system            kube-proxy-4jjg7                             1/1     Running   0          34m
kube-system            kube-scheduler-k8s-master                    1/1     Running   0          34m
kubernetes-dashboard   dashboard-metrics-scraper-79c5968bdc-8vvvn   1/1     Running   0          15m
kubernetes-dashboard   kubernetes-dashboard-7448ffc97b-k52f5        1/1     Running   0          10m
```

现在想要访问 Dashboard 有很多种方式，比较常见的有使用 kubectl proxy，但是我习惯使用 NodePort 方法，因为我是部署在虚拟机，并没有部署在本机。

先创建一个名为 `admin-user`  的服务账户，在 `kubernetes-dashboard` 命名空间中。

```cpp
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
EOF
```

在创建一个角色来分配权限

```cpp
cat <<EOF | kubectl apply -f -
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
EOF
```

```cpp
kubectl -n kubernetes-dashboard describe secret  \
$(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
```

就会得到一个类似如下的返回

```cpp
Name:         admin-user-token-v57nw
Namespace:    kubernetes-dashboard
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: admin-user
              kubernetes.io/service-account.uid: 0303243c-4040-4a58-8a47-849ee9ba79c1

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1066 bytes
namespace:  20 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9......
```

取得 `Token` 备用

执行 `kubectl -n kubernetes-dashboard edit service kubernetes-dashboard` 来修改 Dashboard 的属性

```cpp
spec:
  clusterIP: 10.98.54.192
  clusterIPs:
  - 10.98.54.192
  externalTrafficPolicy: Cluster
  ports:
  - nodePort: 30443 // 新增nodePort
    port: 443
    protocol: TCP
    targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
  sessionAffinity: None
  type: NodePort // 修改类型为nodePort
```

保存即可，这里默认的编辑器是 vim，要学会用它保存一下。

保存后即可使用 ip:30443 来访问 Dashboard

<img src="/post/2021-02-03-install-k8s-on-single-cluser_files/login.png"  align="center" width="700px">

填入 Token 即可进入 Dashboard

以上就是所有内容，可以使用 `kubeadm reset` 来重置，然后重新开始，并且尝试阅读每个yaml文件，理解其中的意思

## 参考资料

1. [Dashboard的安装，README: https://github.com/kubernetes/dashboard/tree/v2.1.0](https://github.com/kubernetes/dashboard/tree/v2.1.0)
2. [Dashboard创建用户指南，README: https://github.com/kubernetes/dashboard/blob/v2.1.0/docs/user/access-control/creating-sample-user.md](https://github.com/kubernetes/dashboard/blob/v2.1.0/docs/user/access-control/creating-sample-user.md)
3. [Dashboard 访问方式: https://github.com/kubernetes/dashboard/blob/v2.1.0/docs/user/accessing-dashboard/README.md](https://github.com/kubernetes/dashboard/blob/v2.1.0/docs/user/accessing-dashboard/README.md)
4. [同类教程：https://www.jianshu.com/p/04f5b9791dc4?from=singlemessage](https://www.jianshu.com/p/04f5b9791dc4?from=singlemessage)