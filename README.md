## 什么是VPP
VPP(Vector Packet Processor)是[FD.io](https://fd.io/)(Fast Data I/O)项目中的一个快速的、可扩展的的L2-L4层网络协议栈。

VPP运行于Linux用户空间(Userspace)，支持x86、ARM、和PowerPC架构，基于Intel的[DPDK](https://www.dpdk.org/)(Dataplane Development Kit)技术实现。

VPP以其高性能、成熟的技术、模块化和丰富的特性而广受欢迎。

VPP支持与OpenStack和Kubernetes集成，网络管理特性包括配置(Configuration)、计数(Counters)、采样(Sampling)等。对于开发者，VPP包括插件扩展(Plugin Extensibility)、高性能事件日志(High-Performance Event-Logging)和多种类型的数据包追踪(Packet Tracing)。开发调试包括完整的符号表(Symbol Tables)和广泛的一致性检查(Consistency Checking)。

VPP应用场景包括虚拟交换(vSwitchs)、虚拟路由(vRouters)、网关(Gateways)、防火墙(Firewalls)和负载均衡器(Load Balancers)等。VPP开箱即用，可以在其基础上构建丰富的网络安全应用软件。

详细信息请点击下边的链接：
* [什么是VPP](https://github.com/penybai/vpp-docs)
  - [标量和矢量数据包处理]()
  - [数据包处理图]()
  - [网络协议栈]()
  - [TCP主机协议栈]()
  - [开发者特性]()
  - [架构和操作系统]()
  - [性能]()
* [开始]()
  - [下载和安装VPP]()
  - [VPP进阶教程]()
  - [用户文档]()
  - [开发者文档](https://github.com/penybai/vpp-docs/blob/master/For-Developers.md)
  - [编写文档]()
* [VPP Wiki、Doxygen和其他链接]()
  - [FD.io主站]()
  - [VPP Wiki]()
  - [源码文档(Doxygen)]()
* [使用案例]()
  - [VPP和容器]()
  - [VPP和Iperf3、Trex]()
  - [VPP和虚拟机]()
  - [VPP和VMware/Vmxnet3]()
  - [VPP和访问控制列表(ACL)]()
  - [VPP在云上]()
  - [VPP作为家庭网关]()
  - [VPP和思科网络插件(Contiv)]()
  - [网络模拟器插件]()
* [故障排除]()
  - [如何提交问题]()
  - [CPU负载/使用]()
* [参考]()
* [相关项目]()
* [关于]()