---
title: CNI (二)
date: 2020-01-19 14:59:21
tags: K8S, Container Network Interface
description: 目前K8s中使用比较多的CNI Plugin插件；简单介绍；
---

# CNI（第二篇）

## 介绍

CNI（容器网络接口）是一个规范SPEC。 如何配置调用： 请查看[CNI SPEC](https://github.com/containernetworking/cni/blob/master/SPEC.md)

CNI包括几部分： 

> golang SDK Lib, 用于集成实现网络通信接口；

> Template，用于生成自定义的CNI插件，标准代码工程；

> 标准Document，包括社区公约、描述、roadmaps、milestone等

## 使用方式

### cnitool 介绍

cnitool是将已经编译完成的的容器网络插件Adapter，添加到VM中或者PM中。

```golang
const (
	EnvCNIPath        = "CNI_PATH"   //CNI Adapter bin文件路径（bin的编译需要去掉cgo）
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
正如上面的例子：`cnitool add myptp /var/run/netns/mytest_network`
> 参数0是cnitool工具名称
> 参数1是操作名称（add、del、check）
> 参数3是加载的配置对象 `myptp.*` 文件对象
> 参数4是 net namespace 名称

解析过程：
```golang
import (
	...
	"github.com/containernetworking/cni/libcni"  // golang lib SDK
)

func main() {
	if len(os.Args) < 4 {  // 判断参数
		usage()
		return
	}

	netdir := os.Getenv(EnvNetDir)  //加载conf文件地址
	if netdir == "" {
		netdir = DefaultNetDir
	}
	netconf, err := libcni.LoadConfList(netdir, os.Args[2])
	if err != nil {
		exit(err)
	}

	... 


	ifName, ok := os.LookupEnv(EnvCNIIfname) // 加载网卡
	if !ok {
		ifName = "eth0"
	}

	...

	netns := os.Args[3]    //获得 new working namespace
	netns, err = filepath.Abs(netns)
	if err != nil {
		exit(err)
	}

	...

	// CNI runtime object
	cninet := libcni.NewCNIConfig(filepath.SplitList(os.Getenv(EnvCNIPath)), nil)
	rt := &libcni.RuntimeConf{
		ContainerID:    fmt.Sprintf("cnitool-%x", ha512.Sum512([]byte(netns))[:10]),
		NetNS:          netns,
		IfName:         ifName,
		Args:           cniArgs,
		CapabilityArgs: capabilityArgs,
	}

	...

	// CNI 具体执行方式
	switch os.Args[1] {
	case CmdAdd:
		result, err := cninet.AddNetworkList(context.TODO(), netconf, rt)
		if result != nil {
			_ = result.Print()
		}
		exit(err)
	case CmdCheck:
		err := cninet.CheckNetworkList(context.TODO(), netconf, rt)
		exit(err)
	case CmdDel:
		exit(cninet.DelNetworkList(context.TODO(), netconf, rt))
	}
}

```

## Golang libcni SDK
SDK核心的interface 接口定义：
```golang
import(
	"github.com/containernetworking/cni/pkg/invoke" //syscall 具体调用
	"github.com/containernetworking/cni/pkg/types" // 网络adapter管理，使用Plugin Chains模型进行关联
	"github.com/containernetworking/cni/pkg/utils" // 公共方法：cni conf file中各个字段ValidCheck
	"github.com/containernetworking/cni/pkg/version" // version compare相关: 0.2.x以下版本兼容；0.3.x 和 现在的0.4.x 版本兼容
)
type CNI interface {
	AddNetworkList(ctx context.Context, net *NetworkConfigList, rt *RuntimeConf) (types.Result, error)
	CheckNetworkList(ctx context.Context, net *NetworkConfigList, rt *RuntimeConf) error
	DelNetworkList(ctx context.Context, net *NetworkConfigList, rt *RuntimeConf) error
	GetNetworkListCachedResult(net *NetworkConfigList, rt *RuntimeConf) (types.Result, error)
	GetNetworkListCachedConfig(net *NetworkConfigList, rt *RuntimeConf) ([]byte, *RuntimeConf, error)

	//以下3个最核心接口
	AddNetwork(ctx context.Context, net *NetworkConfig, rt *RuntimeConf) (types.Result, error)
	CheckNetwork(ctx context.Context, net *NetworkConfig, rt *RuntimeConf) error
	DelNetwork(ctx context.Context, net *NetworkConfig, rt *RuntimeConf) error

	GetNetworkCachedResult(net *NetworkConfig, rt *RuntimeConf) (types.Result, error)
	GetNetworkCachedConfig(net *NetworkConfig, rt *RuntimeConf) ([]byte, *RuntimeConf, error)

	ValidateNetworkList(ctx context.Context, net *NetworkConfigList) ([]string, error)
	ValidateNetwork(ctx context.Context, net *NetworkConfig) ([]string, error)
}

type CNIConfig struct {
	Path     []string
	exec     invoke.Exec
	cacheDir string
}

var _ CNI = &CNIConfig{} // *注意，将CNIConfig{}对象指针都是CNI接口的实现实例；
```

### 核心接口分析
```golang
func (c *CNIConfig) addNetwork(ctx context.Context, name, cniVersion string, net *NetworkConfig, prevResult types.Result, rt *RuntimeConf) (types.Result, error) {
	c.ensureExec()  // 添加数据平台
	pluginPath, err := c.exec.FindInPath(net.Network.Type, c.Path)
	if err != nil {
		return nil, err
	}
	if err := utils.ValidateContainerID(rt.ContainerID); err != nil {
		return nil, err
	}
	if err := utils.ValidateNetworkName(name); err != nil {
		return nil, err
	}
	if err := utils.ValidateInterfaceName(rt.IfName); err != nil {
		return nil, err
	}

	newConf, err := buildOneConfig(name, cniVersion, net, prevResult, rt)
	if err != nil {
		return nil, err
	}

	return invoke.ExecPluginWithResult(ctx, pluginPath, newConf.Bytes, c.args("ADD", rt), c.exec) //执行了Plugin 注入os.Exec
}

type RawExec struct {
	Stderr io.Writer
}

func (e *RawExec) ExecPlugin(ctx context.Context, pluginPath string, stdinData []byte, environ []string) ([]byte, error) {
	stdout := &bytes.Buffer{}
	c := exec.CommandContext(ctx, pluginPath)
	c.Env = environ
	c.Stdin = bytes.NewBuffer(stdinData)
	c.Stdout = stdout
	c.Stderr = e.Stderr
	if err := c.Run(); err != nil {
		return nil, pluginErr(err, stdout.Bytes())
	}

	return stdout.Bytes(), nil
}

func pluginErr(err error, output []byte) error {
	if exitError, ok := err.(*exec.ExitError); ok {
		emsg := types.Error{}
		if len(output) == 0 {
			if len(exitError.Stderr) == 0 {
				emsg.Msg = "netplugin failed with no error message"
			} else {
				emsg.Msg = fmt.Sprintf("netplugin failed: %q", string(exitError.Stderr))
			}
		} else if perr := json.Unmarshal(output, &emsg); perr != nil {
			emsg.Msg = fmt.Sprintf("netplugin failed but error parsing its diagnostic message %q: %v", string(output), perr)
		}
		return &emsg
	}

	return err
}

func (e *RawExec) FindInPath(plugin string, paths []string) (string, error) {
	return FindInPath(plugin, paths)
}
```

将数据接入到Os kernel数据

