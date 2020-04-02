---
title: 【k8s】K8S卷挂载引发的Pod逃逸问题
date: 2019-11-26 13:26:05
categories: k8s
tags:
    - volume
    - k8s
---

### 卷挂载`/var/log`可导致pod逃逸

#### k8s查看容器日志

kubectl提供`kebectl logs <pod_name>`命令查看pod容器的日志，实现原理如下图所示：

{% asset_img kebulet_log_serve.jpg %}

k8s api server 对外提供api `/logs` 查看pod日志，实际是由kubelet提供的服务，kubelet在node上启动一个文件服务器提供日志查询服务，目录为`/var/log`，查询api为`/logs`。如下，访问node(10.0.0.76)上kubelet api `/logs` 可以列出node主机上`/var/log`目录下的所有目录和文件。

```bash
ubuntu@VM-0-76-ubuntu:~$ curl -H 'Authorization: Bearer ihZppLLOVDSCRZS1zFPSwSLHk5odz8zg' -k https://10.0.0.76:10250/logs/
<pre>
<a href="alternatives.log">alternatives.log</a>
<a href="apt/">apt/</a>
<a href="auth.log">auth.log</a>
<a href="bootstrap.log">bootstrap.log</a>
<a href="btmp">btmp</a>
<a href="cloud-init-output.log">cloud-init-output.log</a>
<a href="cloud-init.log">cloud-init.log</a>
<a href="containers/">containers/</a>
<a href="dist-upgrade/">dist-upgrade/</a>
<a href="dmesg">dmesg</a>
<a href="dpkg.log">dpkg.log</a>
<a href="faillog">faillog</a>
<a href="fsck/">fsck/</a>
<a href="installer/">installer/</a>
<a href="journal/">journal/</a>
<a href="kern.log">kern.log</a>
<a href="lxd/">lxd/</a>
<a href="ntpstats/">ntpstats/</a>
<a href="pods/">pods/</a>
<a href="qcloud_action.log">qcloud_action.log</a>
<a href="syslog">syslog</a>
<a href="unattended-upgrades/">unattended-upgrades/</a>
<a href="wtmp">wtmp</a>
</pre>
```

kubelet查询相应pod的日志，实际是访问`/var/log/`目录下对应容器目录下的`0.log`文件(见上图)，该文件本质上是一个符号链接，目标日志文件在`/var/lib/docker/containers`目录下。

```bash
#ls -alh 0.log
0.log -> /var/lib/docker/containers/2acb7d007fad51be32c467fd61639d25b3d04918b26a535735f4fb547d6e1b9d/2acb7d007fad51be32c467fd61639d25b3d04918b26a535735f4fb547d6e1b9d-json.log
```

#### pod逃逸

kubelet查看日志文件支持文符号链接。如果容器内挂载了主机`/var/log`目录，可以通过在容器内创建符号链接到`/`,再利用`/logs`可以访问node主机上的任意文件，造成pod逃逸问题。文章[Kubernetes Pod Escape Using Log Mounts](https://blog.aquasec.com/kubernetes-security-pod-escape-log-mounts)针对该问题进行了pod逃逸的实验，容器内创建符号链接到`/`，通过`logs`api获取宿主机的token或SSH私钥文件实现pod逃逸。

容器内符号链接到`/`实现pod逃逸，需要满足如下条件：

* root用户运行容器
* 主机`/var/log`目录挂载到容器内，且可写；
* pod内角色有访问kubelet`logs` api的权限。

kubelet 10250端口未设置`匿名访问`选项，是需要认证授权才能访问`logs`api接口，而kubelet 10255只读端口，可以不用验证和授权机制，直接访问，但是对下列api做了限制，默认禁止访问，这里面就包括`/logs`api，因此不能通过10255端口绕过`logs`api访问授权问题。

```
"/run/", "/exec/", "/attach/", "/portForward/", "/containerLogs/", "/runningpods/", "/logs/"
```

### Subpath Volume Vulnerability in Kubernetes([CVE-2017-1002101](https://github.com/kubernetes/kubernetes/issues/60813))

[kubernetes官方博客](https://kubernetes.io/blog/2018/04/04/fixing-subpath-volume-vulnerability/)对subpath volume漏洞的原因做了深入分析，具体细节在这里不做过多描述，可查看文章了解。

利用subpath volume漏洞可以逃逸容器内的文件系统访问主机上的任何文件。 容器运行时挂载volume需要知道容器内的路径及对应的主机路径，kubernetes将subpath volume 和base volume同等对待，并没有对subpath volume的路径进行有效性校验，例如子路径（symlink跳转解析后的真实路径）是否在父路径范围下，导致可以利用symlink等特性将主机的根路径(**/**)挂载到容器内。

### Ref

* [Kubernetes中的Volume介绍](https://jimmysong.io/posts/kubernetes-volumes-introduction/)
* [详解 Kubernetes Volume 的实现原理](https://draveness.me/kubernetes-volume)
* [subpath volume fixes(github pr)](https://github.com/kubernetes/kubernetes/pull/61044/files)
