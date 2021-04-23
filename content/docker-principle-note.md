+++
title = "Docker 实现原理笔记"
date = 2021-04-23
+++

本文简要地记录 Docker 的实现原理。

在 Linux 中，实现容器的边界，主要有两种技术 Cgroups 和 Namespace. Cgroups 用于对运行的容器进行资源的限制，Namespace 则会将容器隔离起来，实现边界。

在宿主机上，查看容器内运行的进程，和在宿主机器上直接运行的进程看起来一般无二，但在容器内部，却看不到容器之外的进程。这样看来，**容器只是一种被限制的了特殊进程而已**。

<!-- more -->

#### **容器的隔离：Namespace**
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


##### todo


#### **容器的限制：Cgroups**

todo
#### **容器的文件系统：容器镜像 - rootfs**
todo

<center>
{{ resize_image(path="images/overlay_constructs.jpg", width=480, height=240, op="fit") }}
<image src="/images/overlay_constructs.jpg" />
</center>
<center>Overlay2 结构</center>




参考文章：https://www.cnblogs.com/michael9/p/13039700.html
