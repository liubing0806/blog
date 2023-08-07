---
title: "pod基本概念及其相关用法"
date: 2023-08-07
tags: ["k8s"]
---

# pod
pod是kubernetes的原子调度单位，是一组共享某些资源的容器
具体地说，pod内所有的容器都共享了一个Network Namespace 并且可以共享某一个Volume
容器的本质就是宿主机上的一个进程，通过Namespace做隔离，Cgroups做限制，rootfs做文件系统

pod扮演了传统基础设施中的虚拟机的角色，而容器扮演了虚拟机中的程序

pod中几个重要字段的概念和用法：

## NodeSelector
```
apiVersion:v1
kind:pod
	spec:
		nodeSelector:
			diskType:ssd

```

意味着这个pod只会在有diskType:ssd节点上运行