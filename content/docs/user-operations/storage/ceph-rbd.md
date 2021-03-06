---
title: ceph-rbd块存储
description: 高性能块存储设备
weight: 5015
---

### ceph-rbd 块存储介绍

Ceph 引入了一个新的 RBD 协议，也就是 Ceph 块设备（Ceph Block Device）。RBD 为客户端提供了可靠、分布式、高性能的块存储。RBD 块呈带状分布在多个 Ceph 对象之上，而这些对象本身又分布在整个 Ceph 存储集群中，因此能够保证数据的可靠性以及性能。RBD 已经被 Linux 内核支持，换句话说，RBD 驱动程序在过去的几年里已经很好地跟 Linux 内核集成。几乎所有的 Linux 操作系统发行版都支持 RBD。除了可靠性和性能之外，RBD 也支持其他的企业级特性，例如完整和增量式快照，精简的配置，写时复制（copy-on-write）式克隆，以及其他特性。RBD 还支持全内存式缓存，这可以大大提高它的性能。

Ceph 块设备完全支持云平台，例如 OpenStack、CloudStack 等。在这些云平台中它已经被证明是成功的，并且具有丰富的特性。

> 需用户自己准备 ceph 环境，可参考 [官方安装文档](http://docs.ceph.com/docs/master/start/)

### 安装 ceph 集群

测试环境限制，我们使用 Ubuntu 单节点安装 ceph 集群，安装 ceph 的 jewel 版本，多节点 ceph 集群以及其他版本请参考官方教程。

- ceph 更新源

由于 ceph 在安装过程中，使用默认源速度较慢，这里使用阿里云的源

```bash
echo "deb http://mirrors.aliyun.com/ceph/debian-jewel xenial main" >> /etc/apt/sources.list
apt-get clean
apt-get update
```

- 安装部署程序

官方推荐使用 `ceph-deploy` 执行程序进行 ceph 集群的安装。

```bash
apt-get install ceph-deploy
```

- 创建文件夹，记录 ceph 配置文件

所有的操作均在该文件夹中，该文件夹会记录 ceph 在创建时的配置文件以及 admin 账户的秘钥。

```bash
mkdir ceph-cluster && cd ceph-cluster
```

- 安装监控节点

安装监控节点到 node1

> node1 是主机名，必须能够解析到 node1 主机对应的 IP

```bash
ceph-deploy new node1
```

由于使用单节点安装，需要修改配置文件，设置默认数据池大小为 1

```bash
echo "osd pool default size = 1
osd max object name len = 256
osd max object namespace len = 64
mon_pg_warn_max_per_osd = 2000
mon clock drift allowed = 30
mon clock drift warn backoff = 30
rbd cache writethrough until flush = false" >> ceph.conf
```

- 安装 ceph 服务

```bash
ceph-deploy install node1
```

- 部署监控节点

```bash
ceph-deploy mon create-initial
```

- 准备 ceph 存储路径并准备 osd 服务

将 `/data/ceph` 用作 ceph 的数据共享目录

```bash
mkdir -p /data/ceph/osd
ceph-deploy osd prepare node1:/data/ceph/osd
```

- 激活 osd 服务

```bash
chown -R ceph:ceph /data/ceph/
ceph-deploy osd activate node1:/data/ceph/osd
```

> 如果出现权限问题可尝试进行赋权后重新激活

- 确认 ceph 集群状态

```bash
ceph health
```

结果显示 `HEALTH_OK` 说明集群部署正常。出现异常可以执行 `ceph -s` 确定 ceph 中的异常信息。

如果在某些地方碰到麻烦，想从头再来，可以用下列命令清除配置：

```bash
ceph-deploy purgedata node1
ceph-deploy forgetkeys
```

用下列命令可以连 Ceph 安装包一起清除：

```bash
ceph-deploy purge node1
```

如果执行了 purge ，你必须重新安装 Ceph 。

### 安装驱动

使用 ceph 官方 csi 项目 [ceph-csi](https://github.com/ceph/ceph-csi.git)安装 ceph 驱动。

- 下载项目

```bash
git clone https://github.com/ceph/ceph-csi.git && cd ceph-csi
```

- 创建 rbac 账户

```bash
kubectl create -f csi-provisioner-rbac.yaml
kubectl create -f csi-nodeplugin-rbac.yaml
```

- 准备配置文件

配置文件内容制定了 csi 驱动要使用的 ceph 集群，记录在 ConfigMap 中。测试例子如下，

```bash
$ ceph -s
cluster 9660aec4-16a2-4929-b179-c28cef2b5ab0
health HEALTH_OK
monmap e1: 1 mons at {node1=172.24.203.202:6789/0}
      election epoch 3, quorum 0 node1
osdmap e7: 1 osds: 1 up, 1 in
      flags sortbitwise,require_jewel_osds
pgmap v882: 64 pgs, 1 pools, 7488 kB data, 14 objects
      17357 MB used, 20972 MB / 40188 MB avail
            64 active+clean
```

clusterID 为 ceph 集群的 id，通过`ceph -s`可以看到 cluster 信息
monitors 为 ceph 服务的监控服务地址，对应着 monmap 中的节点信息

```bash
apiVersion: v1
kind: ConfigMap
data:
  config.json: |-
    [
      {
        "clusterID": "9cab3178-b0e1-4d4c-80d6-d1b6cd399cda",
        "monitors": [
          "172.24.203.202:6789"
        ]
      }
    ]
metadata:
  name: ceph-csi-config
```

- 创建 ConfigMap

```bash
kubectl create -f csi-config-map.yaml
```

- 创建驱动服务

```bash
kubectl create -f csi-rbdplugin-provisioner.yaml
kubectl create -f csi-rbdplugin.yaml
```

- 确定服务启动

根据文档安装，最后会有这样的 pod 实例正常运行

```bash
$ kubectl get po
NAME                                         READY   STATUS    RESTARTS   AGE
csi-rbdplugin-provisioner-688c49bd49-7cwcw   6/6     Running   0          59m
csi-rbdplugin-provisioner-688c49bd49-ffbmc   6/6     Running   0          59m
csi-rbdplugin-provisioner-688c49bd49-s4nh2   6/6     Running   0          59m
csi-rbdplugin-qsnlb                          3/3     Running   0          58m
```

使用官方 demo 测试驱动是否正常，官方项目中 `example` 目录下给出了测试用例，使用 nginx 镜像挂载 ceph 存储。

```bash
kubectl create -f secret.yaml
kubectl create -f storageclass.yaml
kubectl create -f pvc.yaml
kubectl create -f pod.yaml
```

确定 nginx 容器是否正常启动

```bash
$ kubectl get po
NAME                                         READY   STATUS    RESTARTS   AGE
csi-rbd-demo-pod                             1/1     Running   0          46m
csi-rbdplugin-provisioner-688c49bd49-7cwcw   6/6     Running   0          59m
csi-rbdplugin-provisioner-688c49bd49-ffbmc   6/6     Running   0          59m
csi-rbdplugin-provisioner-688c49bd49-s4nh2   6/6     Running   0          59m
csi-rbdplugin-qsnlb                          3/3     Running   0          58m
```

如果没有启动成功可以通过，观察 provisioner 组件的日志，确定问题。

```bash
kubectl logs -f csi-rbdplugin-provisioner-688c49bd49-7cwcw -c csi-provisioner
```

> 请注意替换 pod 的名字。 -c 指定 provisoner 实例中的容器，provisoner 实例中会启动 csi-provisioner 实例负责提供存储

### 存储的使用

上面使用官方 demo 时创建了 storageClass 对象，此时 Rainbond 平台会监控 storageClass 的创建，并将其记录到数据库中，用户可以通过 Rainbond 控制台选择 storageClass 对应的存储类型用在有状态组件上。

> 须通过重启组件或者更新组件来触发存储的生效

### 检查结果

存储是否生效可以通过组件是否可以正常启动来判断，组件正常启动则说明组件已经正常挂载了存储，也可以到 ceph 集群中确定存储的情况，确定是否存在对应大小的存储，其状态是否是使用中的状态。
