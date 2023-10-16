---
title: 我想 Debug 容器运行时
date: 2023-10-14 10:08:00
tags:
    - golang
    - containerd
    - docker
    - k8s
    - kubernetes
categories: golang

---

出于好奇，我想弄明白  [Containerd](https://github.com/containerd/containerd) 是如何处理容器 stdio 的，因为我在 k8s 环境中使用 [Containerd](https://github.com/containerd/containerd) 作为容器运行时，且观察到容器中的标准 io 为如下形式：

```shell
/proc/1/fd # ls -l
total 0
lrwx------    1 root     root            64 Oct 10 02:25 0 -> /dev/null
l-wx------    1 root     root            64 Oct 10 02:25 1 -> pipe:[45731]
l-wx------    1 root     root            64 Oct 10 02:25 2 -> pipe:[45732]
```

这是个非常简单的容器，只是往标准输出打印一些内容。从容器进程打开的文件描述符中，我们看到其标准输入被丢弃（通过重定向到 **/dev/null** ），而标准输出和标准错误都被重定向到 **Linux 管道**。

是的，我对 Containerd 如何做到这一点有着浓厚的兴趣，这驱使我产生了 Debug 容器运行时的念头。

这篇短文仅仅介绍我如何达到 Debug 的目的，并不会介绍 Containerd 如何设置标准 io，这部分内容我会放在接下来想写的《终端闲思录》里面。

## Containerd 的生态链位置

首先应该明确一下 Containerd 在整个容器调用链里的位置，我借用一幅容器技术架构图来说明：

![图 1-1 容器技术架构](https://qiniu.liupzmin.com/container.png)

从图中可知，Containerd 通过 CRI 接口上承 k8s，通过 OCI 接口下接 runc，真正创建容器和启动容器的是底层的 runc，所有的 CRI 容器运行时均依赖于 runc 。

你可能已经被 OCI 和 CRI 绕晕了，简言之，这是容器圈里的两大标准：

- **Open Container Initiative (OCI):** a set of standards for containers, describing the image format, runtime, and distribution.
- **Container Runtime Interface (CRI) in Kubernetes:** An API that allows you to use different container runtimes in Kubernetes.

只要记住 OCI 是容器标准，包扩容器创建，镜像格式等等，而 CRI 是属于 k8s 的接口，每个对接 k8s 的容器运行时都要实现这一套接口才能被 kubelet 调用。

## Debug 前的准备

我 Debug 的对象就是实现了 CRI 的 Containerd，但我却面临两个问题：

1. Containerd 需要 root 权限运行，而我并不想在 root 用户下再配置一次 ide 。
2. 虽然可以在 root 下使用 delve 直接调试二进制文件，但我并不清楚 kubelet 调用 Containerd 时传递的参数，而我 Debug 的初衷就是想弄清楚 kubelet 传递的参数内容。

**所以，我决定 Debug 一个运行中的 Containerd！**

这很容易做到，使用 `dlv attach pid`就可以 Debug Containerd 进程。万事俱备，只欠一套 k8s 环境。所幸，我们有 minikube ！

minikube 是在一个虚拟机中运行所有相关组件的，所以有两个问题需要解决：

1. 在 minikube 的虚拟机中准备 delve 工具。
2. 准备一个可调试的 Containerd 服务。

第一点很好解决，minikube 虚拟机和 Host 之间是通过一个网桥通信的，使用`minikube ssh`命令登录到 minikube 虚拟机中，scp 宿主机的工具到 minikube 中就可以了。

通过软件仓库安装的 Containerd 都是经过编译优化的二进制文件，里面缺乏调试信息，因此我们需要编译一份带有调试信息的二进制文件，观察 Containerd 的 makefile 可以发现如下内容：

```makefile
ifndef GODEBUG
	EXTRA_LDFLAGS += -s -w
	DEBUG_GO_GCFLAGS :=
	DEBUG_TAGS :=
else
	DEBUG_GO_GCFLAGS := -gcflags=all="-N -l"
	DEBUG_TAGS := static_build
endif
```

如果环境变量没有设置 GODEBUG ，那么就启用编译优化，否则增加调试信息，所以我们在 make 之前，执行一下`export GODEBUG=1`即可编译出可调试的二进制文件了。编译完成后，使用 scp 拷贝到 minikube 中，重新启动 Containerd 即可。

delve 读取的本地文件系统中项目的内容用于显示单步中的代码，所以，还需要将编译时所用的源码 copy 到 minikube 中，并保持原有路径！

## 开始 Debug

上述准备工作完成后，就可以进入 minikube 虚拟机进行 debug 了：

```bash
richard@Richard-Manjaro:~ » minikube ssh
docker@minikube:~$ sudo -i
root@minikube:~# ps -ef|grep containerd|grep -v shim
root         504       1  3 06:47 ?        00:00:02 /usr/bin/containerd -l debug
root         873       1  3 06:48 ?        00:00:02 /var/lib/minikube/binaries/v1.27.4/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --config=/var/lib/kubelet/config.yaml --container-runtime-endpoint=unix:///run/containerd/containerd.sock --hostname-override=minikube --kubeconfig=/etc/kubernetes/kubelet.conf --node-ip=192.168.49.2
root@minikube:~# dlv attach 504
Type 'help' for list of commands.
(dlv) bp
Breakpoint runtime-fatal-throw (enabled) at 0x559126263aa4,0x559126263b84 for (multiple functions)() <multiple locations>:0 (0)
Breakpoint unrecovered-panic (enabled) at 0x559126263f04 for runtime.fatalpanic() /usr/lib/go/src/runtime/panic.go:1188 (0)
	print runtime.curg._panic.arg
(dlv) b pkg/cri/server/container_create.go:222
Breakpoint 1 set at 0x559127d44f4d for github.com/containerd/containerd/pkg/cri/server.(*criService).CreateContainer() /home/richard/opensource/containerd/pkg/cri/server/container_create.go:222
(dlv) b services/tasks/local.go:167
Breakpoint 2 set at 0x55912734f072 for github.com/containerd/containerd/services/tasks.(*local).Create() /home/richard/opensource/containerd/services/tasks/local.go:167
(dlv) bp
Breakpoint runtime-fatal-throw (enabled) at 0x559126263aa4,0x559126263b84 for (multiple functions)() <multiple locations>:0 (0)
Breakpoint unrecovered-panic (enabled) at 0x559126263f04 for runtime.fatalpanic() /usr/lib/go/src/runtime/panic.go:1188 (0)
	print runtime.curg._panic.arg
Breakpoint 1 (enabled) at 0x559127d44f4d for github.com/containerd/containerd/pkg/cri/server.(*criService).CreateContainer() /home/richard/opensource/containerd/pkg/cri/server/container_create.go:222 (0)
Breakpoint 2 (enabled) at 0x55912734f072 for github.com/containerd/containerd/services/tasks.(*local).Create() /home/richard/opensource/containerd/services/tasks/local.go:167 (0)
(dlv) c
> github.com/containerd/containerd/services/tasks.(*local).Create() /home/richard/opensource/containerd/services/tasks/local.go:167 (hits goroutine(2264):1 total:1) (PC: 0x55912734f072)
   162:		monitor   runtime.TaskMonitor
   163:		v2Runtime runtime.PlatformRuntime
   164:	}
   165:	
   166:	func (l *local) Create(ctx context.Context, r *api.CreateTaskRequest, _ ...grpc.CallOption) (*api.CreateTaskResponse, error) {
=> 167:		container, err := l.getContainer(ctx, r.ContainerID)
   168:		if err != nil {
   169:			return nil, errdefs.ToGRPC(err)
   170:		}
   171:		checkpointPath, err := getRestorePath(container.Runtime.Name, r.Options)
   172:		if err != nil {
(dlv) 
```

进入虚拟机后，切换到 root 执行 delve 指令 `dlv attach 504`，我在此设置了两个断点：

1. pkg/cri/server/container_create.go:222 这是 Containerd 的 CRI 实现里创建容器的接口函数，此函数隶属于一个 grpc server，当我们使用 kubectl 创建工作负载时，会触发此断点。
2. services/tasks/local.go:167 这是 Containerd 将 CRI 请求转化为任务事件的创建入口。

使用 kubectl 提交一个负载就可以单步调试啦 ！O(∩_∩)O~

## 一些注意事项

1. 由图 1-1 可知，k8s 使用 CRI 与容器运行时交互，所以确保 minikube 的运行时使用 Containerd，而不是 docker。docker 虽然也会使用 Containerd，但这种情况下  k8s 不会走 Containerd 的 CRI 接口，相反走的是 docker-shim 的接口，而后者已在 k8s 1.24 中被抛弃。

   ```shell
   richard@Richard-Manjaro:~/playground/k8s » ikube get nodes -o wide
   NAME       STATUS     ROLES           AGE     VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION       CONTAINER-RUNTIME
   minikube   NotReady   control-plane   5d23h   v1.27.4   192.168.49.2   <none>        Ubuntu 22.04.2 LTS   5.15.133-1-MANJARO   containerd://1.7.6
   ```

2. 如果想通过 Containerd 的日志进行简单的 debug，那么开启 debug 日志即可，方法为在执行文件后追加 —log-level=debug

   ```shell
   docker@minikube:~$ cat /usr/lib/systemd/system/containerd.service 
   ......
   
   [Unit]
   Description=containerd container runtime
   Documentation=https://containerd.io
   After=network.target local-fs.target
   
   [Service]
   ExecStartPre=-/sbin/modprobe overlay
   
   ExecStart=/usr/bin/containerd -l debug
   ......
   ```

3. 一定要在你的玩具环境中调试，任何公共环境都不应被用来 debug ！

***参考文献***

1. [The differences between Docker, containerd, CRI-O and runc](https://www.tutorialworks.com/difference-docker-containerd-runc-crio-oci/)
