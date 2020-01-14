## 如何提交问题

## 提交Bugs

尽管每种情况都不尽相同，但本节介绍如何收集数据，这将有助于有效利用每个人处理vpp bugs的时间。

在按下Jira按钮创建错误报告（或发送电子邮件至vpp-dev@lists.fd.io）之前，请问自己是否有足够的信息供他人理解，并通过合理的努力重现该问题。**不建议单独发送邮件给维护者、提交者和项目PTL（项目团队主管）**。

解决明显错误的一个好策略：提交详细的Jira tickets（票证），然后（也许从Jira tickets说明中）将问题的简短描述发送到vpp-dev@lists.fd.io。您可以在发送Jira tickets之前先发送电子邮件至vpp-dev@lists.fd.io询问一些问题。

## BUG报告中包含的数据

### 镜像版本和操作环境

请确保包含vpp镜像版本和命令行参数。
```
$ sudo bash
# vppctl show version verbose cmdline
Version:                  v18.07-rc0~509-gb9124828
Compiled by:              vppuser
Compile host:             vppbuild
Compile date:             Fri Jul 13 09:05:37 EDT 2018
Compile location:         /scratch/vpp-showversion
Compiler:                 GCC 7.3.0
Current PID:              5211
Command line arguments:
  /scratch/vpp-showversion/build-root/install-vpp_debug-native/vpp/bin/vpp
  unix
  interactive
```

关于操作环境：如果涉及特定VM/容器/裸机环境的不当行为，请详细描述该环境：
* Linux发行版（例如Ubuntu 18.04.2 LTS，CentOS-7等）
* NIC类型（ixgbe，i40e，enic等等），vhost-user，tuntap
* NUMA配置（如果适用）

请注意CPU架构（x86_86，aarch64）和硬件平台。

如果可行，请针对已发布（released）的软件或未修改的master/latest软件报告问题。

### show命令输出

每种情况都不同。如果问题涉及调试CLI命令序列，请启用CLI命令日志记录，并发送涉及的序列。请注意，调试CLI是开发人员的工具-不作任何明示或暗示的担保-并且我们可能选择不修复调试CLI错误。

请包括“show error”[错误计数器]输出。通常，“clear error”，发送少量流量然后“show error”通常很有用，尤其是在嘈杂的网络上运行vpp时。

请包括ip4/ip6/mpls FIB内容（“show ip fib”，“show ip6 fib”，“show mpls fib”，“show mpls tunnel”）。

请包括“show hardware”，“show interface”和“show interface address”输出。

这是一组综合的命令，通常在发送流量之前/之后很有用。发送流量之前：
```
vppctl clear hardware
vppctl clear interface
vppctl clear error
vppctl clear run
```

发送一些流量，然后发送如下命令：

```
vppctl show version verbose
vppctl show hardware
vppctl show interface address
vppctl show interface
vppctl show run
vppctl show error
```

这是一些特定于协议的show命令，这些命令也可能有意义。仅包括已配置的功能。
```
vppctl show l2fib
vppctl show bridge-domain

vppctl show ip fib
vppctl show ip neighbors

vppctl show ip6 fib
vppctl show ip6 neighbors

vppctl show mpls fib
vppctl show mpls tunnel
```

### 网络拓扑

请提供对网络拓扑的清晰描述，包括L2/IP/MPLS/网段路由寻址详细信息。如果您希望人们重现和调试问题，那么这是必须的。

在某种或更高的拓扑复杂性水平下，重现原始设置变得很成问题。

### 数据包跟踪器输出

如果您捕获了似乎相关的数据包跟踪器（Packet Tracer）输出，请包括在内。

```
vppctl trace add dpdk-input 100  # or similar
```

发送流量，然后查看跟踪信息：

```
vppctl show trace
```

## 采集事后数据

它应该不言而喻，但是无论如何：请将事后数据放在明显且容易接近的地方。尝试获取帐户，凭据和IP地址所浪费的时间只会延迟解决问题的时间。

请记住将事后数据位置信息添加到Jira tickets（票证）中。

### Syslog输出

vpp信号处理程序通常在退出前在/var/log/syslog中写入一定数量的数据。确保检查这些证据，例如通过“grep/usr/bin/vpp /var/log/syslog”或类似文件。

### 二进制API跟踪

如果问题涉及控制平面API消息序列-甚至是很长的序列-请启用控制平面API跟踪。控制平面API事后跟踪最终在/tmp /api_post_mortem.<pid>中。

请记住将事后二进制api跟踪数据放在可访问的地方。

这些API跟踪数据在vpp引擎将流量扔到地板上的情况下尤其有用，例如，缺少默认路由或其他相似的问题。

确保在vpp启动配置文件/etc/vpp/startup.conf中保留默认节“…api-trace {on}…”，或将其包含在业务流程软件传递的命令行参数中。

### Core文件

生产系统以及长期运行的生产前浸泡测试（soak-test）系统必须安排收集core镜像。有多种方法来配置core镜像捕获，包括例如Ubuntu的“corekeeper”软件包。必要时，以下非常基本的序列将捕获/tmp/dumps中可用的vpp core文件。

```
# mkdir -p /tmp/dumps
# sysctl -w debug.exception-trace=1
# sysctl -w kernel.core_pattern="/tmp/dumps/%e-%t"
# ulimit -c unlimited
# echo 2 > /proc/sys/fs/suid_dumpable
```

如果从systemd启动VPP，则还需要编辑/lib/systemd/system/vpp.service并取消注释“LimitCORE = infinity”行，然后重新启动VPP。

Vpp核心文件通常看起来很大，但总是稀疏。Gzip将它们压缩到可管理的大小。一个多GB的core文件通常压缩为10-20MB。

解压缩vpp core文件时，我们建议使用如图所示的“dd”来创建一个稀疏的，未压缩的core文件：
```
$ zcat vpp_core.gz | dd conv=sparse of=vpp_core
```

请记住将压缩的core文件放在可访问的位置。

确保在vpp启动配置文件/etc/vpp/startup.conf中保留默认节“…unix {…full-coredump…}…”，或将其包含在业务流程软件传递的命令行参数中。

## 私有镜像的Core文件

私有镜像中的core文件需要特殊处理。如果需要这样做，请将与core文件相对应的确切Debian软件包（或RPM）复制到与core文件相同的公共位置。这是一个没有任何借口的硬性要求。

尤其是：

```
libvppinfra_<version>_<arch>.deb # vppinfra library
libvppinfra-dev_<version>_<arch>.deb # vppinfra library development pkg
vpp_<version>_<arch>.deb         # the vpp executable
vpp-dbg_<version>_<arch>.deb     # debug symbols
vpp-dev_<version>_<arch>.deb     # vpp development pkg
vpp-lib_<version>_<arch>.deb     # shared libraries
vpp-plugin-core_<version>_<arch>.deb # core plugins
vpp-plugin-dpdk_<version>_<arch>.deb # dpdk plugin
```

作为参考，请在Jira票证中包括git commit-ID，branch和git repo信息（除了gerrit.fd.io以外）。

请注意，git commit-ids是head[latest]合并补丁的加密总和。他们对本地工作空间的修改，分支或相关的git repo都一无所获。

即使给出了逐字节的相同源代码树，也可以轻松构建截然不同的二进制工件（artifacts），只需要一个不同的工具链版本。

### 动态core文件压缩

根据操作要求，可以在生成core文件时对其进行压缩。请注意，动态压缩vpp core文件需要花费几秒钟的时间，在此期间所有数据包处理活动都将暂停。

要动态创建压缩的core文件，请创建以下脚本，例如 在/usr/local/bin/compressed_corefiles中，由root拥有，可执行文件：
```
#!/bin/sh
exec /bin/gzip -f - >"/tmp/dumps/core-$1.$2.gz"
```

如下调整内核core文件模式：
```
sysctl -w kernel.core_pattern="|/usr/local/bin/compressed_corefiles %e %t"
```

### Core文件摘要

底线：请按照信函中的core文件处理说明进行操作，并不复杂。只需将与core文件相对应的确切Debian软件包或RPM复制到可访问的位置。

如果我们在设置过程中发现映像文件和core文件不匹配，则只会延迟问题的解决；更不用说激怒那些浪费时间的人了。