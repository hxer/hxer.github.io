---
title: 【k8s】Kubeadm安装k8s支持secret加密
date: 2019-10-15 14:39:36
categories: k8s
tags:
    - secret
    - k8s
---

<!--more-->

* 增加k8s yum源，编辑`/etc/yum.repos.d/kubernetes.repo`

  ```
  [kubernetes]
  name=Kubernetes Repository
  baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
  enable=1
  gpgcheck=0
  ```

* 安装kubeadm和相关工具

  ```bash
  yum install -y kubelet kubeadm kubectl --disbaleexcludes=kubernetes
  ```

* 启动docker和kubelet

  ```bash
  systemctl enable docker && systemctl staart docker
  systemctl enable kubelet && systemctl staart kubelet
  ```

* 编辑kubeadm init配置文件

  ```bash
  kubeadm config print init-defaults > init.default.yaml
  ```

  参考init.defaukt.yaml文件编辑配置文件init.config.yaml

  ```yaml
  apiVersion: kubeadm.k8s.io/v1beta1
  imageRepository: gcr.akscn.io/google_containers
  kind: ClusterConfiguration
  kubernetesVersion: v1.14.0
  networking:
    dnsDomain: cluster.local
    podSubnet: ""
    serviceSubnet: 10.96.0.0/12
  apiServer:
    extraArgs:
      anonymous-auth: "false"
      encryption-provider-config: /etc/kubernetes/pki/kube-secret.yaml
  ```

  * k8s版本14, api-server支持secret加密参数为`encryption-provider-config`，参考：https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#before-you-begin

  * kube-secret.yaml文件

    ```yaml
    apiVersion: apiserver.config.k8s.io/v1
    kind: EncryptionConfiguration
    resources:
      - resources:
        - secrets
        providers:
        - aescbc:
            keys:
            - name: key1
              secret: BhgJ6ldAAvHHdOkE9gGmMQj5seDc3nHeyQ+NOpZjeyY=
        - identity: {}
    ```

  * kube-secret.yaml路径选择

    kube-secret.yaml需要mount到api-server pod可访问的路径，这样api-server才能访问，这里选择默认会mount的路径`/etc/kubernetes/pki`

* 拉取相关镜像

  ```bash
  kubeadm config images pull --config=init-config.yaml
  ```

* 安装master

  ```bash
  kubeadm init --config=init-config.yaml
  ```

* 验证aescbc加密

  * 安装etcdctl

    ```shell
    #!/bin/bash
    ETCD_VER=v3.3.10
    ETCD_DIR=etcd-download
    DOWNLOAD_URL=https://github.com/coreos/etcd/releases/download
    
    # Download
    mkdir ${ETCD_DIR}
    cd ${ETCD_DIR}
    wget ${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz
    tar -xzvf etcd-${ETCD_VER}-linux-amd64.tar.gz
    
    # install
    cd etcd-${ETCD_VER}-linux-amd64
    cp etcdctl /usr/local/bin/
    ```

  * 验证

    ```bash
    ETCDCTL_API=3 etcdctl get /registry/secrets/default/default-token-6q4bn --cacert="/etc/kubernetes/pki/etcd/ca.crt" --cert="/etc/kubernetes/pki/apiserver-etcd-client.crt" --key="/etc/kubernetes/pki/apiserver-etcd-client.key" --endpoints=127.0.0.1:2379
    ```
