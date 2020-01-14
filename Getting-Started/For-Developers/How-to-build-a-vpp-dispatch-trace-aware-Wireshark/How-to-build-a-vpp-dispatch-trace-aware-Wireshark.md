## 如何给Wireshark编译一个VPP调度跟踪器
vpp pcap调度跟踪剖析器已合并到Wireshark主分支中，因此过程很简单。下载wireshark，进行编译并安装。

### 下载Wireshark源码
Wrieshark源码仓库很大，因此可能需要花费一些时间来克隆。
```
    git clone https://code.wireshark.org/review/wireshark
```

### 安装必备软件包
除了Ubuntu 18.04系统上通常安装的软件包之外，以下列出了一些必需的软件包才能编译Wireshark：
```
    libgcrypt11-dev flex bison qtbase5-dev qttools5-dev-tools qttools5-dev
    qtmultimedia5-dev libqt5svg5-dev libpcap-dev qt5-default
```

### 编译Wireshark
幸运的是，Wireshark使用cmake，因此至少在Ubuntu 18.04上相对容易编译。
```
    $ cd wireshark
    $ mkdir build
    $ cd build
    $ cmake -G Ninja ../
    $ ninja -j 8
    $ sudo ninja install
```

### 配置一个pcap调度跟踪
配置vpp以某种方式通过流量，然后配置：
```
    vpp# pcap dispatch trace on max 10000 file vppcapture buffer-trace dpdk-input 1000
```

或者类似的，运行流量一段时间，确保捕获到一些数据，然后保存调度跟踪如下：

```
    vpp# pcap dispatch trace off
```

### 在Wireshark中显示
在启用了vpp的wireshark中显示/tmp/vppcapture。运气好的话，Wireshark的普通版本将拒绝处理vpp调度跟踪pcap文件，因为它们不了解encap类型。

设置wireshark使用vpp.bufferindex进行过滤，以观察单个数据包遍历转发图的情况。否则，您会看到一个ip4-lookup的vector，然后是ip4-rewrite的vector，等等。