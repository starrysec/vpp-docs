## VPP配置-CLI和startup.conf

成功安装后，VPP将在/etc/vpp/目录中安装名为startup.conf的启动配置文件。可以定制此文件以使VPP根据需要运行，文件默认有典型的默认值。

以下是有关此文件及其包含的一些参数和值的更多详细信息。

### 命令行参数
在我们描述启动配置文件（startup.conf）的详细信息之前，应该提到的是，可以在没有启动配置文件的情况下启动VPP。

参数按节名称分组。当为一个节提供多个参数时，该节的所有参数都必须用花括号括起来。例如，要通过命令行使用节名称为“unix”的配置数据启动VPP：
```

```

### 启动配置文件(startup.conf)

### 配置参数
#### unix部分
#### api-trace部分
#### api-segment部分
#### socksvr部分
#### cpu部分
#### buffers部分
#### dpdk部分
#### plugins部分

### 一些高级配置参数
#### acl-plugin部分
#### api-queue部分
#### cj部分
#### dns部分
#### heapsize部分
#### ip部分
#### ip6部分
#### l2learn部分
#### l2tp部分
#### logging部分
#### mactime部分

### "map"参数
#### nat部分 
#### oam部分
#### physmem部分
#### tapcli部分
#### tcp部分
#### tls部分
#### tuntap部分
#### vhost-user部分
#### vlib部分