## VPP和访问控制列表(ACL)

本节概述了可用于在FD.io VPP中实现ACL的选项。由于有很多方法可以解决类似ACL的功能，因此值得对这些选项进行单独调研，并对功能和性能进行一些评论。

本文档中的所有性能编号和示例复用了FD.io CSIT v19.04性能报告。对于FD.io VPP 19.04版本，所有信息和性能都是准确的。下面的部分性能和操作数据与FD.io CSIT性能报告中的那些部分直接相关。

### 摘要

| 选项 | 相对性能 | 特点和说明 |
| ------ | ------ | ------ |
| FD.io VPP ACL插件 | 最低 | 严格匹配L2-L4字段，支持有状态和无状态 |
| FD.io VPP COP | 最高（仅软件） | 匹配L3 IP，支持无状态 |
| FD.io VPP Flow | 最高（加速后） | 严格匹配L2-L4字段，支持无状态，流（flows）数目有限制 |
| FD.io VPP Classifiers | 待定 | 配置数据包前80字节中的任何字段，暂未测量 |

### FD.io VPP ACL选项

#### FD.io VPP ACL插件

该插件最初是作为FD.io VPP和OpenStack集成的一部分开发的。需要在特定接口上启用该插件。

##### 支持下列字段的有状态和无状态ACL

* MAC地址
* IP地址
* UDP端口
* TCP端口和标志
* ICMP消息

##### 方向性

* Input ACLs
  - 在IP流分类之前运行。
* ACLs
  - 在接口输出之前运行。

##### 动作

* 允许(无状态)
* 丢弃(有状态)
* 允许+反射(有状态)

##### 有状态(sf，Stateful)

* 动作：允许+反射
* 最严格的优化，以及最常见的用例。
* 由于有状态使用流缓存，因此速度更快，这意味着ACL命中仅被获取一次，在流的前面，仅变为查找。
* 使用更多的内存，确定性较低，因为流缓存使其更容易受到内存层次结构和位置的影响。

##### 无状态（sl，Stateless）

* 动作：允许，丢弃
* 优化程度较低，使用案例较少。
* 由于没有流高速缓存，因此速度较慢，每个新数据包都会产生相同数量的ACL处理。
* 使用更少的内存，并且更具确定性（与有状态相比）。

### 操作数据

#### 输入/无状态
#### 测试用例：10ge2p1x520-ethip4udp-ip4base-iacl1sl-10kflows-ndrpdr
```
DUT1:
Thread 0 vpp_main (lcore 1)
Time 3.8, average vectors/node 0.00, last 128 main loops 0.00 per node 0.00
  vector rates in 0.0000e0, out 0.0000e0, drop 0.0000e0, punt 0.0000e0
             Name                 State         Calls          Vectors        Suspends         Clocks       Vectors/Call
acl-plugin-fa-cleaner-process   any wait                 0               0              14          1.29e3            0.00
acl-plugin-fa-worker-cleaner-pinterrupt wa               7               0               0          9.18e2            0.00
api-rx-from-ring                 active                  0               0              52          8.96e4            0.00
dpdk-process                    any wait                 0               0               1          1.35e4            0.00
fib-walk                        any wait                 0               0               2          2.69e3            0.00
ip6-icmp-neighbor-discovery-ev  any wait                 0               0               4          1.32e3            0.00
lisp-retry-service              any wait                 0               0               2          2.90e3            0.00
unix-epoll-input                 polling              7037               0               0          1.25e6            0.00
vpe-oam-process                 any wait                 0               0               2          2.28e3            0.00

Thread 1 vpp_wk_0 (lcore 2)
Time 3.8, average vectors/node 249.02, last 128 main loops 32.00 per node 273.07
  vector rates in 6.1118e6, out 6.1118e6, drop 0.0000e0, punt 0.0000e0
             Name                 State         Calls          Vectors        Suspends         Clocks       Vectors/Call
TenGigabitEtherneta/0/0-output   active              47106        11721472               0          9.47e0          248.83
TenGigabitEtherneta/0/0-tx       active              47106        11721472               0          4.22e1          248.83
TenGigabitEtherneta/0/1-output   active              47106        11721472               0          1.02e1          248.83
TenGigabitEtherneta/0/1-tx       active              47106        11721472               0          4.18e1          248.83
acl-plugin-fa-worker-cleaner-pinterrupt wa               7               0               0          1.39e3            0.00
acl-plugin-in-ip4-fa             active              94107        23442944               0          1.75e2          249.11
dpdk-input                       polling             47106        23442944               0          4.64e1          497.66
ethernet-input                   active              94212        23442944               0          1.55e1          248.83
ip4-input-no-checksum            active              94107        23442944               0          3.23e1          249.11
ip4-lookup                       active              94107        23442944               0          2.91e1          249.11
ip4-rewrite                      active              94107        23442944               0          2.48e1          249.11
unix-epoll-input                 polling                46               0               0          1.54e3            0.00
```

#### 输入/有状态
##### 测试用例：64b-1t1c-ethip4udp-ip4base-iacl1sf-10kflows-ndrpdr
```
DUT1:
Thread 0 vpp_main (lcore 1)
Time 3.9, average vectors/node 0.00, last 128 main loops 0.00 per node 0.00
  vector rates in 0.0000e0, out 0.0000e0, drop 0.0000e0, punt 0.0000e0
             Name                 State         Calls          Vectors        Suspends         Clocks       Vectors/Call
acl-plugin-fa-cleaner-process   any wait                 0               0              16          1.40e3            0.00
acl-plugin-fa-worker-cleaner-pinterrupt wa               8               0               0          8.97e2            0.00
api-rx-from-ring                 active                  0               0              52          7.12e4            0.00
dpdk-process                    any wait                 0               0               1          1.69e4            0.00
fib-walk                        any wait                 0               0               2          2.55e3            0.00
ip4-reassembly-expire-walk      any wait                 0               0               1          1.27e4            0.00
ip6-icmp-neighbor-discovery-ev  any wait                 0               0               4          1.09e3            0.00
ip6-reassembly-expire-walk      any wait                 0               0               1          2.57e3            0.00
lisp-retry-service              any wait                 0               0               2          1.18e4            0.00
statseg-collector-process       time wait                0               0               1          6.38e3            0.00
unix-epoll-input                 polling              6320               0               0          1.41e6            0.00
vpe-oam-process                 any wait                 0               0               2          7.53e3            0.00

Thread 1 vpp_wk_0 (lcore 2)
Time 3.9, average vectors/node 252.74, last 128 main loops 32.00 per node 273.07
  vector rates in 7.5833e6, out 7.5833e6, drop 0.0000e0, punt 0.0000e0
             Name                 State         Calls          Vectors        Suspends         Clocks       Vectors/Call
TenGigabitEtherneta/0/0-output   active              58325        14738944               0          9.41e0          252.70
TenGigabitEtherneta/0/0-tx       active              58325        14738944               0          4.32e1          252.70
TenGigabitEtherneta/0/1-output   active              58323        14738944               0          1.02e1          252.71
TenGigabitEtherneta/0/1-tx       active              58323        14738944               0          4.31e1          252.71
acl-plugin-fa-worker-cleaner-pinterrupt wa               8               0               0          1.62e3            0.00
acl-plugin-in-ip4-fa             active             116628        29477888               0          1.01e2          252.75
dpdk-input                       polling             58325        29477888               0          4.63e1          505.41
ethernet-input                   active             116648        29477888               0          1.53e1          252.71
ip4-input-no-checksum            active             116628        29477888               0          3.21e1          252.75
ip4-lookup                       active             116628        29477888               0          2.90e1          252.75
ip4-rewrite                      active             116628        29477888               0          2.48e1          252.75
unix-epoll-input                 polling                57               0               0          2.39e3            0.00
```

#### 输出/无状态
##### 测试用例：64b-1t1c-ethip4udp-ip4base-oacl10sl-10kflows-ndrpdr
```
DUT1:
 Thread 0 vpp_main (lcore 1)
 Time 3.8, average vectors/node 0.00, last 128 main loops 0.00 per node 0.00
   vector rates in 0.0000e0, out 0.0000e0, drop 0.0000e0, punt 0.0000e0
              Name                 State         Calls          Vectors        Suspends         Clocks       Vectors/Call
 acl-plugin-fa-cleaner-process   any wait                 0               0              14          1.43e3            0.00
 acl-plugin-fa-worker-cleaner-pinterrupt wa               7               0               0          9.23e2            0.00
 api-rx-from-ring                 active                  0               0              52          8.01e4            0.00
 dpdk-process                    any wait                 0               0               1          1.59e6            0.00
 fib-walk                        any wait                 0               0               2          6.81e3            0.00
 ip6-icmp-neighbor-discovery-ev  any wait                 0               0               4          2.81e3            0.00
 lisp-retry-service              any wait                 0               0               2          3.64e3            0.00
 unix-epoll-input                 polling              4842               0               0          1.81e6            0.00
 vpe-oam-process                 any wait                 0               0               1          2.24e4            0.00

 Thread 1 vpp_wk_0 (lcore 2)
 Time 3.8, average vectors/node 249.29, last 128 main loops 36.00 per node 271.06
   vector rates in 5.9196e6, out 5.9196e6, drop 0.0000e0, punt 0.0000e0
              Name                 State         Calls          Vectors        Suspends         Clocks       Vectors/Call
 TenGigabitEtherneta/0/0-output   active              45595        11363584               0          9.22e0          249.23
 TenGigabitEtherneta/0/0-tx       active              45595        11363584               0          4.25e1          249.23
 TenGigabitEtherneta/0/1-output   active              45594        11363584               0          9.75e0          249.23
 TenGigabitEtherneta/0/1-tx       active              45594        11363584               0          4.21e1          249.23
 acl-plugin-fa-worker-cleaner-pinterrupt wa               7               0               0          1.28e3            0.00
 acl-plugin-out-ip4-fa            active              91155        22727168               0          1.78e2          249.32
 dpdk-input                       polling             45595        22727168               0          4.64e1          498.46
 ethernet-input                   active              91189        22727168               0          1.56e1          249.23
 interface-output                 active              91155        22727168               0          1.13e1          249.32
 ip4-input-no-checksum            active              91155        22727168               0          1.95e1          249.32
 ip4-lookup                       active              91155        22727168               0          2.88e1          249.32
 ip4-rewrite                      active              91155        22727168               0          3.53e1          249.32
 unix-epoll-input                 polling                44               0               0          1.53e3            0.00
```

#### 输出/有状态
##### 测试用例：64b-1t1c-ethip4udp-ip4base-oacl10sf-10kflows-ndrpdr
```
DUT1:
 Thread 0 vpp_main (lcore 1)
 Time 3.8, average vectors/node 0.00, last 128 main loops 0.00 per node 0.00
   vector rates in 0.0000e0, out 0.0000e0, drop 0.0000e0, punt 0.0000e0
              Name                 State         Calls          Vectors        Suspends         Clocks       Vectors/Call
 acl-plugin-fa-cleaner-process   any wait                 0               0              16          1.47e3            0.00
 acl-plugin-fa-worker-cleaner-pinterrupt wa               8               0               0          8.51e2            0.00
 api-rx-from-ring                 active                  0               0              50          7.24e4            0.00
 dpdk-process                    any wait                 0               0               2          1.93e4            0.00
 fib-walk                        any wait                 0               0               2          2.02e3            0.00
 ip4-reassembly-expire-walk      any wait                 0               0               1          3.96e3            0.00
 ip6-icmp-neighbor-discovery-ev  any wait                 0               0               4          9.84e2            0.00
 ip6-reassembly-expire-walk      any wait                 0               0               1          3.76e3            0.00
 lisp-retry-service              any wait                 0               0               2          1.49e4            0.00
 statseg-collector-process       time wait                0               0               1          4.98e3            0.00
 unix-epoll-input                 polling              5653               0               0          1.55e6            0.00
 vpe-oam-process                 any wait                 0               0               2          1.90e3            0.00

 Thread 1 vpp_wk_0 (lcore 2)
 Time 3.8, average vectors/node 250.85, last 128 main loops 36.00 per node 271.06
   vector rates in 7.2686e6, out 7.2686e6, drop 0.0000e0, punt 0.0000e0
              Name                 State         Calls          Vectors        Suspends         Clocks       Vectors/Call
 TenGigabitEtherneta/0/0-output   active              55639        13930752               0          9.33e0          250.38
 TenGigabitEtherneta/0/0-tx       active              55639        13930752               0          4.27e1          250.38
 TenGigabitEtherneta/0/1-output   active              55636        13930758               0          9.81e0          250.39
 TenGigabitEtherneta/0/1-tx       active              55636        13930758               0          4.33e1          250.39
 acl-plugin-fa-worker-cleaner-pinterrupt wa               8               0               0          1.62e3            0.00
 acl-plugin-out-ip4-fa            active             110988        27861510               0          1.04e2          251.03
 dpdk-input                       polling             55639        27861510               0          4.62e1          500.76
 ethernet-input                   active             111275        27861510               0          1.55e1          250.38
 interface-output                 active             110988        27861510               0          1.21e1          251.03
 ip4-input-no-checksum            active             110988        27861510               0          1.95e1          251.03
 ip4-lookup                       active             110988        27861510               0          2.89e1          251.03
 ip4-rewrite                      active             110988        27861510               0          3.55e1          251.03
 unix-epoll-input                 polling                54               0               0          2.43e3            0.00
```

### 性能

| 测试用例 | MPPS | 时钟周期/每个数据包 |
| ------ | ------ | ------ |
| ethip4-ip4base | 18.26 | 136 |
| ethip4ip4udp-ip4base-iacl1sl-10kflows | 9.134 | 273 |
| ethip4ip4udp-ip4base-iacl1sf-10kflows | 11.06 | 226 |

### Input ACLS (SKX)
![](https://github.com/penybai/vpp-docs/blob/master/images/ip4-2n-iacl.png)

### Output ACLs (HSW)
![](https://github.com/penybai/vpp-docs/blob/master/images/ip4-3n-oacl.png)

### 配置
#### 有状态
```
$ sudo vppctl ip_add_del_route 20.20.20.0/24 via 1.1.1.2 sw_if_index 1 resolve-attempts 10 count 1
$ sudo vppctl acl_add_replace ipv4 permit src 30.30.30.1/32 dst 40.40.40.1/32 sport 1000 dport 1000, ipv4 permit+reflect src 10.10.10.0/24, ipv4 permit+reflect src 20.20.20.0/24
$ sudo vppctl acl_interface_set_acl_list sw_if_index 2 input 0
$ sudo vppctl acl_interface_set_acl_list sw_if_index 1 input 0
```

#### 无状态
```
$ sudo vppctl ip_add_del_route 20.20.20.0/24 via 1.1.1.2 sw_if_index 1 resolve-attempts 10 count 1
$ sudo vppctl acl_add_replace ipv4 permit src 30.30.30.1/32 dst 40.40.40.1/32 sport 1000 dport 1000, ipv4 permit src 10.10.10.0/24, ipv4 permit src 20.20.20.0/24
$ sudo vppctl acl_interface_set_acl_list sw_if_index 2 input 0
$ sudo vppctl acl_interface_set_acl_list sw_if_index 1 input 0
```

#### 链接
* [FD.io安全组概述（Security Groups overview）]()
* [反射ACL（Reflexive Access Control Lists）]()
* [Andrew Yuort关于ACL的博客]()

### FD.io VPP COP
### FD.io VPP Flow
### FD.io VPP Classifiers

FD.io VPP中最灵活的ACL形式，使用户能够匹配数据包头的前80个字节中的任何位置。

#### 配置
匹配一个IPv6...
```
$ sudo vppctl classify table mask l3 ip6 dst buckets 64
$ sudo vppctl classify session hit-next 0 table-index 0 match l3 ip6 dst 2001:db8:1::2 opaque-index 42
$ sudo vppctl set interface l2 input classify intfc host-s0_s1 ip6-table 0
```

#### 链接
* [分类器概述（Overview of classifiers）]()
* [FD.io分类器概述（FD.io Classifiers Overview）]()
* [FD.io分类器CLI（FD.io Classifiers CLI）]()
* [Andrew Yourt的示例代码]()