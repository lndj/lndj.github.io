+++
title = "Docker 实现原理笔记"
date = 2021-04-23
+++

本文简要地记录 Docker 的实现原理。

在 Linux 中，实现容器的边界，主要有两种技术 Cgroups 和 Namespace. Cgroups 用于对运行的容器进行资源的限制，Namespace 则会将容器隔离起来，实现边界。

在宿主机上，查看容器内运行的进程，和在宿主机器上直接运行的进程看起来一般无二，但在容器内部，却看不到容器之外的进程。这样看来，**容器只是一种被限制的了特殊进程而已**。

<!-- more -->

#### **一、容器的隔离：Namespace**
```shell 
docker run --rm -it busybox
```
进入容器之后执行，查看容器内的进程信息：
```shell
$ ps
PID  USER     TIME  COMMAND
1    root     0:00  sh
9    root     0:00  ps
```
可以看到第一个进程的 `pid` 为 1， 该进程的 `pid` 为 `1747375`。其实就是 `Linux` 的 `Namespace` 机制。

`Linux` 下，可以使用 `clone` 来创建一个进程，指定 `CLONE_NEWPID` 参数，就会创建一个全新的进程空间，函数签名：

```c
int pid = clone(main_function, stack_size, CLONE_NEWPID | SIGCHLD, NULL); 
```
下面列出一下相关的参数：

| 分类               | 系统调用参数  | 相关内核版本  |
|------------------|---------------|------------ |
| Mount namespaces | CLONE_NEWNS   | Linux 2.4.19|
| UTS namespaces   | CLONE_NEWUTS  | Linux 2.6.19|
| IPC namespaces   | CLONE_NEWIPC  | Linux 2.6.19|
| PID namespaces   | CLONE_NEWPID  | Linux 2.6.24|
| Network namespaces | CLONE_NEWNET | 始于Linux 2.6.24 完成于 Linux 2.6.29|
| User namespaces  | CLONE_NEWUSER | 始于 Linux 2.6.23 完成于 Linux 3.8|

<br />

>  PS: `Linux` 下和进程创建相关的函数： `clone` `fork` `vfork`

#### **二、容器的限制：Cgroups**

上述的 `Namespace` 技术，实现了容器和宿主机、容器和容器之间的隔离，但是他们之间还是公用系统资源的，如果一个容器占用了大量的系统资源，就会导致其他的容器被影响。**`Cgroups` 技术是 `Linux` 内核中用于对进程设置资源限制的技术**。

> Linux Cgroups 全称是 Linux Control Group，主要的作用就是限制进程组使用的资源上限，包括 CPU，内存，磁盘，网络带宽。还可以对进程进行优先级设置，审计，挂起和恢复等操作。

在当前的大多数 `Linux` 发行版中，我们可以使用 `systemctl` 来管理 `cgroup`。

> 针对 `systemd` 的一些使用，参阅： [http://www.ruanyifeng.com/blog/2016/03/systemd-tutorial-commands.html](http://www.ruanyifeng.com/blog/2016/03/systemd-tutorial-commands.html)

`Systemd` 可以管理所有系统资源。不同的资源统称为 `Unit`（单位）。`Unit` 分 12 种：

| 类型    | 作用                              |
|--------|-----------------------------------|
|Service |一个服务或者一个应用，具体定义在配置文件中|
|Target|多个 `Unit` 构成的一个组|
|Device|硬件设备|
|Mount|文件系统的挂载点|
|Automount|自动挂载点|
|Path|文件或路径|
|Scope|不是由 `Systemd` 启动的外部进程|
|Slice|进程组|
|Snapshot|`Systemd` 快照，可以切回某个快照|
|Socket|进程间通信的 `socket`|
|Swap|`swap` 文件|
|Timer|定时器|

可以操作，创建一个临时的 `cgroup`，对其进行资源的限制：

```shell
 # 创建一个叫 top-test 的服务，在名为 test 的 slice 中运行
[root@localhost ~]# systemd-run --unit=top-test --slice=test top -b
Running as unit top-test.service.
```

执行上述命令后，`top-test` 服务就已经在后台开始运行了。我们可以使用 `systemctl-cgls` 查看所有的 `Cgroups`，也可以使用 `systemctl status top-test` 来查看服务运行状态。

可以使用 `cat /proc/{pid}/cgroup` 来查看当前的 `cgroup` 信息。

然后，对其进行资源限制操作：

```shell
$ systemctl set-property top-test.service CPUShares=800 MemoryLimit=600M
```

再次去查看 `cgroup` 信息，会发现在 `cpu` 和 `memory` 追加了一些内容。

这时可以在 `/sys/fs/cgroup/memory/test.slice` 和 `/sys/fs/cgroup/cpu/test.slice` 目录下，多出了一个叫 `top-test.service` 的目录。查看其中 `toptest.service/cpu.shares` 的内容，可以看到 `CPU` 被限制到了 `800`。

在 `Docker` 中，我们也可以做这样限制：

```shell
$ docker run -it --cpu-period=100000 --cpu-quota=20000 ubuntu /bin/bash
```

关于 `docker` 具体的限制，可以在 `sys/fs/cgroup/cpu/docekr/` 等文件夹来查看。

#### **三、容器的文件系统：容器镜像 - rootfs**

在容器内，应该看到完全独立的文件系统，而且不会受到宿主机以及其他容器的影响。这个独立的文件系统，就叫做容器镜像。它还有一个更专业的名字叫 `rootfs`. `rootfs` 中包含了一个操作系统所需要的文件，配置和目录，但并不包含系统内核。 因为在 `Linux` 中，文件和内核是分开存放的，操作系统只有在开启启动时才会加载指定的内核。这也就意味着，**所有的容器都会共享宿主机上操作系统的内核**。

Docker 最早的 slogan 是 `Build once, run anywhere` ，有了 `rootfs` ，这个问题就被很好的解决了。因为在镜像内，打包的不仅仅是应用，还有所需要的依赖，都被封装在一起。这就解决了无论是在哪，应用都可以很好的运行的原因。

不光如此，`rootfs` 还解决了可重用性的问题，想象这个场景，你通过 `rootfs` 打包了一个包含 `java` 环境的 `centos` 镜像，别人需要在容器内跑一个 `apache` 的服务，那么他是否需要从头开始搭建 `java` 环境呢？`docker` 在解决这个问题时，引入了一个叫层的概念，每次针对 `rootfs` 的修改，都只保存增量的内容，而不是 `fork` 一个新镜像。

层级的想法，同样来自于 `Linux`，一个叫 `union file system` （联合文件系统）。它最主要的功能就是将不同位置的目录联合挂载到同一个目录下。对应在 `Docker` 里面，不同的环境则使用了不同的联合文件系统。比如 centos7 下最新的版本使用的是 `overlay2`，而 `Ubuntu 16.04` 和 `Docker CE 18.05` 使用的是 `AuFS`。
> 详情参见： [Docker storage drivers](https://docs.docker.com/storage/storagedriver/select-storage-driver/)

可以通过 `docker info` 来查询使用的存储驱动。

<center>
<image src="/images/overlay_constructs.jpg" />
</center>
<center>Overlay2 整体结构图</center>

Docker 一般的存储位置在 `/var/lib/docker`，内部存储的结构可以实现了 Docker 的镜像和镜像的分层。

> 详细情况 todo
#### **四、容器的网络**

todo


#### **五、其他总结**

**对资源的限制方面：**

由于 Docker 内资源的限制通过 `Cgroup` 实现，而 `Cgroup` 有很多不完善的地方，比如
对 `/proc` 的处理问题。进入容器后，执行 `top` 命令，看到的信息和宿主机是一样的，而不是配置后的容器的数据。（可以通过 `lxcfs` 修正）。
在运行 `java` 程序时，给容器内设置的内存为 `4g`，使用默认的 `jvm` 配置。而默认的 `jvm` 读取的内存是宿主机（可能大于 `4g`），这样就会出现 `OOM` 的情况。

**在隔离性方面：**

因为本身上容器就是一种进程，而所有的进程都需要共享一个系统内核，因此：

- 在 `Windows` 上运行 `Linux` 容器，或者 `Linux` 宿主机运行高版本内核的容器就无法实现。
- 在 `Linux` 内核中，有许多资源和对象不能 `Namespace` 化，如时间，比如通过 `settimeofday(2)` 系统调用 修改时间，整个宿主机的实际都会被修改。
- 安全的问题，共享宿主机内核的事实，容器暴露出的攻击面更大。

<br />

---

#### 参考文章：

- [https://www.cnblogs.com/michael9/p/13039700.html](https://www.cnblogs.com/michael9/p/13039700.html)
- [https://coolshell.cn/articles/17010.html](https://coolshell.cn/articles/17010.html)
- [http://www.ruanyifeng.com/blog/2016/03/systemd-tutorial-commands.html](http://www.ruanyifeng.com/blog/2016/03/systemd-tutorial-commands.html)
