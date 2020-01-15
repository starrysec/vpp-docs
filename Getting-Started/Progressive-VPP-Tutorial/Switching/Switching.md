## 交换

### 可以学习到的技能

在这个练习中您将学习到这些新技能：
1. 将一个接口关联到一个桥域（bridge domain）
2. 创建一个loopback接口
3. 为桥域创建一个桥虚拟接口（BVI, Bridge Virtual Interface）
4. 检查桥域

### 练习学习到的命令
1. [show bridge]()
2. [show bridge detail]()
3. [set int l2 bridge]()
4. [show l2fib verbose]()

### 拓扑
![](https://github.com/penybai/vpp-docs/blob/master/images/switching_topology.jpeg)

***Switching Topology***

### 初始状态

与以前的练习不同，对于此练习，您要开始进行表格绘制。

注意：您将丢失FD.io VPP实例中的所有现有配置！

要从以前的练习中清除现有配置，请运行：

```
$ ps -ef | grep vpp | awk '{print $2}'| xargs sudo kill
$ sudo ip link del dev vpp1host
$ # do the next command if you are cleaning up from this example
$ sudo ip link del dev vpp1vpp2
```

### 运行FD.io VPP实例

### vpp1连接主机

### vpp1连接vpp2

### 配置vpp1上的桥域

### 配置vpp2上的loopback接口

### 配置vpp2上的桥域

### 从主机ping vpp和从vpp ping主机

### 检查l2 fib
```
vpp# show l2fib verbose
Mac Address     BD Idx           Interface           Index  static  filter  bvi   Mac Age (min)
de:ad:00:00:00:00    1            host-vpp1vpp2           2       0       0     0      disabled
c2:f6:88:31:7b:8e    1            host-vpp1out            1       0       0     0      disabled
2 l2fib entries
```

```
vpp# show l2fib verbose
Mac Address     BD Idx           Interface           Index  static  filter  bvi   Mac Age (min)
de:ad:00:00:00:00    1                loop0               2       1       0     1      disabled
c2:f6:88:31:7b:8e    1            host-vpp2vpp1           1       0       0     0      disabled
2 l2fib entries
```