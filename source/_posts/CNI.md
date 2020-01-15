---
title: CNI
date: 2020-01-15 10:45:57
tags: K8S, Container Network Interface
description: Container Network Interface - networking for Linux containers
---

# CNI

## 介绍

CNI（容器网络接口）是一个规范和库，用于编写用于在Linux容器中配置网络接口的插件以及许多受支持的插件组成。

CNI包括几部分： 

> golang SDK Lib, 用于集成实现网络通信接口；

> Template，用于生成自定义的CNI插件，标准代码工程；

> 标准Document，包括社区公约、描述、roadmaps、milestone等

## 使用方式

### cnitool 介绍

cnitool是将已经编译完成的的容器网络插件Adapter，添加到VM中或者PM中。

```golang
const (
	EnvCNIPath        = "CNI_PATH"   //CNI Adapter bin文件路径（bin的编译需要去掉ggo）
	EnvNetDir         = "NETCONFPATH"  //部署（添加、删除、检查）adapter需要的json配置文件路径
	EnvCapabilityArgs = "CAP_ARGS"  // CAP 参数
	EnvCNIArgs        = "CNI_ARGS"  // adapter 外部传递的参数，一般不用，将args放在json文件中
	EnvCNIIfname      = "CNI_IFNAME"  // 设置的容器网卡名称，如eth0

	DefaultNetDir = "/etc/cni/net.d" // 默认CNI 插件路径

	// 以下部分是 插件 添加方式
	CmdAdd   = "add"
	CmdCheck = "check"
	CmdDel   = "del"
)
```

### 插件使用

准备二进制插件：
```bash
go get github.com/containernetworking/plugins
go build ptp -o myptp
```


准备网络配置：
```bash
echo `
{
	"cniVersion": "0.4.0",
	"name": "myptp",
	"type": "ptp",  //veth pair
	"ipMasq": true,
	"ipam": {
		"type": "host-local",
		"subnet": "172.16.29.0/24",
		"routes": [{
			"dst": "0.0.0.0/0"
		}]
	}
}` > /etc/cni/net.d/myptp.conflist
```

准备好配置后部署容器网络：

第一步：创建网络Namespace， 添加名称为mytest_network
```bash
sudo ip netns add mytest_network
```

第二步：添加容器网络myptp
```bash
sudo CNI_PATH=/etc/cni/net.d/ cnitool add myptp /var/run/netns/mytest_network
```

第三步：调测网络
```bash
sudo ip -n mytest_network addr
sudo ip netns exec mytest_network ping -c 1 4.2.2.2
```

最后，清理网络
```bash
sudo CNI_PATH=./etc/cni/net.d/ cnitool del myptp /var/run/netns/mytest_network
sudo ip netns del mytest_network
```

### cnitool调用插件



