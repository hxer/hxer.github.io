---
title: 【k8s】源码调试k8s
date: 2019-09-24 13:30:19
categories: k8s
tags:
    - debug
    - k8s
---

虚拟机搭建k8s源码编译调试环境，宿主机使用golang IDE远程连接进行调试。

<!--more-->

### 虚拟机配置调试环境

虚拟机内存设置大于4G，内存过小可能导致源码编译失败。操作系统：Centos7

#### 安装必要软件

参考[官方文档](https://github.com/kubernetes/community/blob/master/contributors/devel/running-locally.md)，在虚拟机中安装必要的软件

* Docker

  ```bash
  yum install -y yum-utils device-mapper-persistent-data lvm2
  yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
  yum install -y docker-ce
  systemctl enable docker
  systemctl start docker
  ```

* etcd

  ```bash
  yum install -y etcd
  ```

* openssl

  ```bash
  yum install -y openssl
  ```

* go

  [k8s go兼容列表](https://github.com/kubernetes/community/blob/master/contributors/devel/development.md#go-versions)

  ```bash
  wget https://dl.google.com/go/go1.12.7.linux-amd64.tar.gz
  tar -C /usr/local -xzf go1.12.7.linux-amd64.tar.gz
  mkdir $HOME/go
  echo "
  export GOROOT=/usr/local/go
  export GOPATH=$HOME/go
  export PATH=$PATH:$GOROOT/bin
  " >> ~/.bashrc
  
  source ~/.bashrc
  ```

  * 环境变量
    * GOROOT: 指向golang的安装路径
    * GOPATH: 指向golang的工作空间

* cfssl

  ```bash
  go get -u -v github.com/cloudflare/cfssl/cmd/...
  ```
  
  创建软链接避免源码编译时找不到cfssl，`ln -s $GOPATH/bin/cfssl $GOROOT/bin/cfssl`。

#### 安装delve调试工具

```bash
go get -u -v github.com/go-delve/delve/cmd/dlv
```

创建软链接

```bash
ln -s $GOPATH/bin/dlv $GOROOT/bin/dlv
```



#### 源码编译

* 获取代码

  ```
  git clone https://github.com/kubernetes/kubernetes.git $GOPATH/src/k8s.io/kubernetes
  cd $GOPATH/src/k8s.io/kubernetes
  git checkout -b v1.10.3 v1.10.3  # 切换到v1.10.3分支
  ```


* 单机集群启动

  参考[这里](https://github.com/kubernetes/community/blob/master/contributors/devel/running-locally.md)，直接执行`hack/local-up-cluster.sh`脚本即可启动一个轻量的k8s集群，但由于 `local-up-cluster.sh` 默认使用的是 `hyperkube` 镜像在容器中启动 k8s 服务，这样不方便使用dlv调试代码。

  使用[local-debug-cluster.sh](https://gist.github.com/hxer/8f38c725614d5af3317d1a6bc1db36a9)启动单集群，直接编译出带调试信息的可执行文件，并在主机中启动k8s相关进程。`local-debug-cluster.sh`文件主要是修改`hyperkube`为对应的组件，如下所示：

  ```shell
  #local hyperkube_path=$(kube::util::find-binary "hyperkube")
  local hyperkube_path=$(kube::util::find-binary "kube-apiserver")
  
  ...
  
  if [ "x$GO_OUT" == "x" ]; then
      #make -C "${KUBE_ROOT}" WHAT="cmd/kubectl cmd/hyperkube"
      make -C "${KUBE_ROOT}" GOGCFLAGS="-N -l" WHAT="cmd/kubectl cmd/kube-proxy cmd/kube-apiserver cmd/kube-controller-manager cmd/cloud-controller-manager cmd/kube-scheduler cmd/kubelet"
  else
      echo "skipped the build."
  fi
  
  ...
  
  #${CONTROLPLANE_SUDO} "${GO_OUT}/hyperkube" apiserver ${swagger_arg} ${audit_arg} ${authorizer_arg} ${priv_arg} ${runtime_config} \
  ${CONTROLPLANE_SUDO} "${GO_OUT}/kube-apiserver" ${swagger_arg} ${audit_arg} ${authorizer_arg} ${priv_arg} ${runtime_config} \
      
  ...
  
  ```

  编译并启动集群

  ```javascript
  ./hack/local-debug-cluster.sh
  ```

  第一次执行上面的命令编译并启动集群，后面修改了某个模块的代码，执行以下代码即可单独编译该模块：

  ```javascript
  make GOGCFLAGS="-N -l" WHAT="cmd/kube-apiserver" # 假设只编译kube-apiserver这一个模块
  ```

  执行下面的命令直接启动集群（跳过编译）

  ```javascript
  ./hack/local-debug-cluster.sh -O
  ```

### 调试k8s代码

#### 虚拟机dlv启动k8s

正常启动k8s单机集群后，ps看一下，可以到启动了7个进程：

```javascript
ps -ef|grep kube|grep -v grep
```

假设要调试kube-apiserver这个进程，则`先查出kube-apiserver启动时的运行参数`：

```javascript
ps -ef|grep kube|grep -v grep

root      29540  12913 13 20:55 pts/0    00:01:26 /root/go/src/k8s.io/kubernetes/_output/local/bin/linux/amd64/kube-apiserver --authorization-mode=Node,RBAC --runtime-config=admissionregistration.k8s.io/v1alpha1,settings.k8s.io/v1alpha1 --cloud-provider= --cloud-config= --v=3 --vmodule= --cert-dir=/var/run/kubernetes --client-ca-file=/var/run/kubernetes/client-ca.crt --service-account-key-file=/tmp/kube-serviceaccount.key --service-account-lookup=true --enable-admission-plugins=Initializers,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota,PodPreset,StorageObjectInUseProtection --disable-admission-plugins= --admission-control-config-file= --bind-address=0.0.0.0 --secure-port=6443 --tls-cert-file=/var/run/kubernetes/serving-kube-apiserver.crt --tls-private-key-file=/var/run/kubernetes/serving-kube-apiserver.key --insecure-bind-address=127.0.0.1 --insecure-port=8080 --storage-backend=etcd3 --etcd-servers=http://127.0.0.1:2379 --service-cluster-ip-range=10.0.0.0/24 --feature-gates=AllAlpha=false --external-hostname=localhost --requestheader-username-headers=X-Remote-User --requestheader-group-headers=X-Remote-Group --requestheader-extra-headers-prefix=X-Remote-Extra- --requestheader-client-ca-file=/var/run/kubernetes/request-header-ca.crt --requestheader-allowed-names=system:auth-proxy --proxy-client-cert-file=/var/run/kubernetes/client-auth-proxy.crt --proxy-client-key-file=/var/run/kubernetes/client-auth-proxy.key --cors-allowed-origins=/127.0.0.1(:[0-9]+)?$,/localhost(:[0-9]+)?$
```

可以看到该进程的PID是29540，于是停掉该进程，并用dlv启动：

```javascript
kill -15 29540
dlv --listen=:2333 --headless=true --api-version=2 exec /root/go/src/k8s.io/kubernetes/_output/local/bin/linux/amd64/kube-apiserver -- --authorization-mode=Node,RBAC --runtime-config=admissionregistration.k8s.io/v1alpha1,settings.k8s.io/v1alpha1 --cloud-provider= --cloud-config= --v=3 --vmodule= --cert-dir=/var/run/kubernetes --client-ca-file=/var/run/kubernetes/client-ca.crt --service-account-key-file=/tmp/kube-serviceaccount.key --service-account-lookup=true --enable-admission-plugins=Initializers,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota,PodPreset,StorageObjectInUseProtection --disable-admission-plugins= --admission-control-config-file= --bind-address=0.0.0.0 --secure-port=6443 --tls-cert-file=/var/run/kubernetes/serving-kube-apiserver.crt --tls-private-key-file=/var/run/kubernetes/serving-kube-apiserver.key --insecure-bind-address=127.0.0.1 --insecure-port=8080 --storage-backend=etcd3 --etcd-servers=http://127.0.0.1:2379 --service-cluster-ip-range=10.0.0.0/24 --feature-gates=AllAlpha=false --external-hostname=localhost --requestheader-username-headers=X-Remote-User --requestheader-group-headers=X-Remote-Group --requestheader-extra-headers-prefix=X-Remote-Extra- --requestheader-client-ca-file=/var/run/kubernetes/request-header-ca.crt --requestheader-allowed-names=system:auth-proxy --proxy-client-cert-file=/var/run/kubernetes/client-auth-proxy.crt --proxy-client-key-file=/var/run/kubernetes/client-auth-proxy.key --cors-allowed-origins='/127.0.0.1(:[0-9]+)?$,/localhost(:[0-9]+)?$'
```

`--cors-allowed-origins`参数值涉及特殊字符，需要引号包裹。

允许防火墙允许2333端口接入：

```javascript
firewall-cmd --zone=public --add-port=2333/tcp --permanent
firewall-cmd --reload
```

#### 宿主机IDE远程连接

配置go语言环境，拉取k8s源码，切换和虚拟机相同的版本。

然后在goland里新建一个远程调试任务，这里的`10.211.55.3`为虚拟机的IP，`39999`为dlv的远程调试端口。在`kubernetes/cmd/kube-apiserver/apiserver.go`的main方法处打上断点后，启动该远程调试任务，即可正常进行断点调试了。

{% asset_img debug-api-server-conf.png %}

{% asset_img debug-api-server.png %}

#### dlv调试不同模块

* kubelet

  ```bash
  dlv -l :2334 --headless=true --api-version=2 exec /root/go/src/k8s.io/kubernetes/_output/local/bin/linux/amd64/kubelet -- --v=3 --vmodule= --chaos-chance=0.0 --container-runtime=docker --rkt-path= --rkt-stage1-image= --hostname-override=127.0.0.1 --cloud-provider= --cloud-config= --address=127.0.0.1 --kubeconfig /var/run/kubernetes/kubelet.kubeconfig --feature-gates=AllAlpha=false --cpu-cfs-quota=true --enable-controller-attach-detach=true --cgroups-per-qos=true --cgroup-driver= --keep-terminated-pod-volumes=true --eviction-hard='memory.available<100Mi,nodefs.available<10%,nodefs.inodesFree<5%' --eviction-soft= --eviction-pressure-transition-period=1m --pod-manifest-path=/var/run/kubernetes/static-pods --fail-swap-on=false --cluster-dns=10.0.0.10 --cluster-domain=cluster.local --port=10250
  ```
  
  在`kubernetes/cmd/kubelet/kubelet.go`的main方法处打上断点后，同上启动远程调试任务。
  
  {% asset_img debug-api-kubelet.png %}
  
### 集群安装异常排查

* 集群启动脚本(local-up-cluster.sh)获取cgroup driver信息不准导致kubelet启动失败

  kubelet cgroup driver会根据docker使用的cgroup driver信息，设置相同的cgroup driver.如下:

  ```shell
  CGROUP_DRIVER=$(docker info | grep "Cgroup Driver:" | cut -f3- -d' ')
  ```

  在docker 19版本中得到的结果为`Driver: cgroupfs`，而不是`cgroupfs`，这是由于`Cgroup Driver:`行前面有空格导致的，需要去除前后空格，[local-debug-cluster.sh](https://gist.github.com/hxer/8f38c725614d5af3317d1a6bc1db36a9)修改如下：

  ```shell
  CGROUP_DRIVER=$(docker info | grep "Cgroup Driver:" | awk '{sub(/^ */, "");sub(/ *$/, "")}1'| cut -f3- -d' ')
  ```

  续：该问题官方已解决https://github.com/hxer/kubernetes/blob/master/hack/local-up-cluster.sh

  ```bash
  CGROUP_DRIVER=$(docker info | grep "Cgroup Driver:" |  sed -e 's/^[[:space:]]*//'|cut -f3- -d' ')
  ```

  

* cgroup driver不一致

  kubelet cgroup driver为systemd，修改docker文件驱动为`systemd`，和kubelet文件驱动保持一致，否则会导致kubelet启动失败，参考：[kubernetes集群的安装异常汇总](https://juejin.im/post/5bbf7dd05188255c652d62fe)

  ```bash
  #修改daemon.json
  vim /etc/docker/daemon.json
  
  #添加如下属性
  "exec-opts": [
      "native.cgroupdriver=systemd"
  ]
  
  # 重启docker
  systemctl daemon-reload
  systemctl restart docker
  ```

  

### 参考

- [搭建k8s的开发调试环境](https://cloud.tencent.com/developer/article/1402449)
- [跟我学 K8S--代码: 调试 Kubernetes](https://segmentfault.com/a/1190000015807702)
- [远程 Debug kubeadm](https://mritd.me/2018/11/25/kubeadm-remote-debug/)
- [kubernetes源码调试体验](http://aspirer.wang/?p=1226)

- [Debug a Go Application in Kubernetes from IDE](https://itnext.io/debug-a-go-application-in-kubernetes-from-ide-c45ad26d8785)