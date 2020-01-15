## VPP配置-CLI和startup.conf

成功安装后，VPP将在/etc/vpp/目录中安装名为startup.conf的启动配置文件。可以定制此文件以使VPP根据需要运行，文件默认有典型的默认值。

以下是有关此文件及其包含的一些参数和值的更多详细信息。

### 命令行参数
在我们描述启动配置文件（startup.conf）的详细信息之前，应该提到的是，可以在没有启动配置文件的情况下启动VPP。

参数按节名称分组。当为一个节提供多个参数时，该节的所有参数都必须用花括号括起来。例如，要通过命令行使用节名称为“unix”的配置数据启动VPP：
```
$ sudo /usr/bin/vpp unix { interactive cli-listen 127.0.0.1:5002 }
```

命令行可以显示为单个字符串或多个字符串。在解析之前，命令行上给出的所有内容都用空格连接成单个字符串。VPP应用程序必须能够找到其自己的可执行镜像。确保其工作的最简单方法是在启动时通过提供其绝对路径来调用VPP应用程序。例如：‘/usr/bin/vpp <options>’，VPP应用程序首先解析它们自己的ELF部分，以创建初始化、配置和退出处理程序（exit handlers）的列表。

使用VPP开发时，在gdb中通常足以启动这样的应用程序：

```
(gdb) run unix interactive
```

### 启动配置文件(startup.conf)

将启动配置指定为VPP的更典型方法是使用启动配置文件（startup.conf）。

通过命令行将该文件的路径提供给VPP应用程序，通常配置文件位于/etc/vpp/startup.conf中，如果VPP作为软件包安装，则默认的startup.conf文件也位于此。

配置文件的格式是一个简单的文本文件，其内容与命令行相同。

一个非常简单的startup.conf文件：
```
$ cat /etc/vpp/startup.conf
unix {
  nodaemon
  log /var/log/vpp/vpp.log
  full-coredump
  cli-listen localhost:5002
}

api-trace {
  on
}

dpdk {
  dev 0000:03:00.0
}
```

VPP通过-c选项加载配置文件，例如：
```
$ sudo /usr/bin/vpp -c /etc/vpp/startup.conf
```

### 配置参数

以下是一些节名称及其相关参数的列表。这不是一个详尽的列表，但是应该能使您了解如何配置VPP。

对于所有配置参数，请在源代码中搜索VLIB_CONFIG_FUNCTION和VLIB_EARLY_CONFIG_FUNCTION的实例。

例如，调用“VLIB_CONFIG_FUNCTION（foo_config，“foo”）”将使函数“foo_config”接收名为“foo”的参数块中给出的所有参数：“foo {arg1 arg2 arg3…}”。

#### unix节

配置VPP启动和行为类型属性，以及任何基于OS的属性。

```
unix {
  nodaemon
  log /var/log/vpp/vpp.log
  full-coredump
  cli-listen /run/vpp/cli.sock
  gid vpp
}
```

##### nodaemon

不要派生/后台运行vpp进程。从进程监视器调用VPP应用程序时，通常使用这种情况。默认情况下，在默认的“startup.conf”文件中进行设置。

```
nodaemon
```

##### interactive

将CLI附加到stdin/stdout，提供一个命令行调试接口。

```
interactive
```

##### log <filename>

启动配置和后续所有CLI命令都会记录到该文件中，尤其在人们不记得或不愿意在bug报告中包含CLI命令的情况下非常有用。默认的“startup.conf”中的日志文件是“/var/log/vpp/vpp.log”文件。

在VPP 18.04中，默认日志文件位置已从“/tmp/vpp.log”移至“/var/log/vpp/vpp.log”。VPP代码与文件位置无关，但是，如果启用了SELinux，则需要新位置才能正确标记文件。请在您的系统上检查本地“startup.conf”文件的位置。

```
log /var/log/vpp/vpp-debug.log
```

##### exec | startup-config <filename>

从文件名读取启动配置。文件的内容将像在CLI上输入的那样执行。这两个关键字是同一功能的别名。如果两者都指定，则只有最后一个才会生效。

这个文件中的CLI命令可能类似于：

```
$ cat /usr/share/vpp/scripts/interface-up.txt
set interface state TenGigabitEthernet1/0/0 up
set interface state TenGigabitEthernet1/0/1 up
```

参数举例：

```
startup-config /usr/share/vpp/scripts/interface-up.txt
```

##### gid <number | name>

设置调用进程有效的组id和组名称。

```
gid vpp
```

##### full-coredump

要求Linux内核转储(dump)所有内存映射的地址区域，而不仅仅是text+data+bss。

```
full-coredump
```

##### coredump-size unlimited | <n>G | <n>M | <n>K | <n>

设置coredump文件的最大大小。输入值可以以GB，MB，KB或字节设置，也可以设置为“无限制”。

```
coredump-size unlimited
```

##### cli-listen <ipaddress:port> | <socket-path>

绑定CLI以侦听地址localhost上的TCP端口5002。接受ipaddress:port形式或文件系统路径；在后一种情况下，将打开本地Unix套接字。默认的“startup.conf”文件是打开套接字“/run/vpp/cli.sock”。

```
cli-listen localhost:5002
cli-listen /run/vpp/cli.sock
```

##### cli-line-mode

在stdin上禁用逐字符I/O。与emacs M-x gud-gdb结合使用时很有用。

```
cli-line-mode
```

##### cli-prompt <string>

配置CLI提示符字符串。

```
cli-prompt vpp-2
```

##### cli-history-limit <n>
##### cli-no-banner
##### cli-no-pager
##### cli-pager-buffer-limit <n>
##### runtime-dir <dir>
##### poll-sleep-usec <n>
##### pidfile <filename>

#### api-trace节
#### api-segment节
#### socksvr节
#### cpu节
#### buffers节
#### dpdk节
#### plugins节

### 一些高级配置参数
#### acl-plugin节
#### api-queue节
#### cj节
#### dns节
#### heapsize节
#### ip节
#### ip6节
#### l2learn节
#### l2tp节
#### logging节
#### mactime节

### "map"参数
#### nat节 
#### oam节
#### physmem节
#### tapcli节
#### tcp节
#### tls节
#### tuntap节
#### vhost-user节
#### vlib节