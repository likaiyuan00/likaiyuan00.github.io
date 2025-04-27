---
title: 使用kubekey快速安装k8s
date: 2025-04-27 11:16:46
tags:
categories: k8s
---
官方地址
https://github.com/kubesphere/kubekey

# 安装
> curl -sfL https://get-kk.kubesphere.io | sh -
## 单节点测试使用
```shell
kk create cluster
#默认 v1.23.17
--with-kubernetes v1.24.1 
#默认docker
--container-manager containerd
#如果不使用--with-kubesphere默认不安装；默认版本为 v3.4.1
--with-kubesphere
```
## 多节点
```shell
kk create config -f deploy.yml
#-f 指定配置文件开始安装
kk create cluster -f deploy.yml
#deploy.yml;其他节点的ip用户名密码的修改成实际的
apiVersion: kubekey.kubesphere.io/v1alpha2
kind: Cluster
metadata:
  name: sample
spec:
  hosts:
  - {name: node1, address: 172.16.0.2, internalAddress: 172.16.0.2, user: ubuntu, password: "Qcloud@123"}
  - {name: node2, address: 172.16.0.3, internalAddress: 172.16.0.3, user: ubuntu, password: "Qcloud@123"}
  roleGroups:
    etcd:
    - node1
    control-plane: 
    - node1
    worker:
    - node1
    - node2
  controlPlaneEndpoint:
    ## Internal loadbalancer for apiservers 
    # internalLoadbalancer: haproxy

    domain: lb.kubesphere.local
    address: ""
    port: 6443
  kubernetes:
    version: v1.23.17
    clusterName: cluster.local
    autoRenewCerts: true
    containerManager: docker
  etcd:
    type: kubekey
  network:
    plugin: calico
    kubePodsCIDR: 10.233.64.0/18
    kubeServiceCIDR: 10.233.0.0/18
    ## multus support. https://github.com/k8snetworkplumbingwg/multus-cni
    multusCNI:
      enabled: false
  registry:
    privateRegistry: ""
    namespaceOverride: ""
    registryMirrors: []
    insecureRegistries: []
  addons: []
----------------------------------------------------
#默认不安装kubesphere需要指定--with-kubesphere
kk create config --with-kubesphere -f deploy-with.yml
```
# 新增删除
```shell
#新增节点接入集群
kk add nodes -f  deploy.yml
#删除节点
kk delete node <nodeName> -f deploy.yml
#删除集群
kk delete cluster [-f deploy.yml]
```

# 升级集群
```shell
使用指定版本升级集群。

kk upgrade [--with-kubernetes version] [--with-kubesphere version] 
仅支持升级 Kubernetes。
仅支持升级 KubeSphere。
支持升级 Kubernetes 和 KubeSphere。
多节点
使用指定的配置文件升级集群。

kk upgrade [--with-kubernetes version] [--with-kubesphere version] [(-f | --filename) path]
如果指定了--with-kubernetes或--with-kubesphere，配置文件也将被更新。
用于-f指定为集群创建而生成的配置文件。
```

# 更新集群证书
>#默认一年<br>
kk  certs renew
![alt text](image.png)
