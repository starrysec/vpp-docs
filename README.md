## 什么是VPP
VPP(Vector Packet Processor)是[FD.io](https://fd.io/)(Fast Data I/O)项目中的一个快速的、可扩展的的L2-L4层网络协议栈。

VPP运行于Linux用户空间(Userspace)，支持x86、ARM、和PowerPC架构，基于Intel的[DPDK](https://www.dpdk.org/)(Dataplane Development Kit)技术实现。

VPP以其高性能、成熟的技术、模块化和丰富的特性而广受欢迎。

VPP支持与OpenStack和Kubernetes集成，网络管理特性包括配置(Configuration)、计数(Counters)、采样(Sampling)等。对于开发者，VPP包括插件扩展(Plugin Extensibility)、高性能事件日志(High-Performance Event-Logging)和多种类型的数据包追踪(Packet Tracing)。开发调试包括完整的符号表(Symbol Tables)和广泛的一致性检查(Consistency Checking)。

VPP应用场景包括虚拟交换(vSwitchs)、虚拟路由(vRouters)、网关(Gateways)、防火墙(Firewalls)和负载均衡器(Load Balancers)等。VPP开箱即用，可以在其基础上构建丰富的网络安全应用软件。

详细信息请点击下边的链接：
* [什么是VPP](https://github.com/penybai/vpp-docs)--Completed
  - [标量和矢量数据包处理](https://github.com/penybai/vpp-docs/blob/master/The-Vector-Packet-Processor/Scalar-vs-Vector-packet-processing.md)--Completed
  - [数据包处理图](https://github.com/penybai/vpp-docs/blob/master/The-Vector-Packet-Processor/The-Packet-Processing-Graph.md)--Completed
  - [网络协议栈](https://github.com/penybai/vpp-docs/blob/master/The-Vector-Packet-Processor/Network-Stack.md)--Completed
  - [TCP主机栈](https://github.com/penybai/vpp-docs/blob/master/The-Vector-Packet-Processor/TCP-Host-Stack.md)--Completed
  - [开发者特性](https://github.com/penybai/vpp-docs/blob/master/The-Vector-Packet-Processor/Features-for-Developers.md)--Completed
  - [架构和操作系统](https://github.com/penybai/vpp-docs/blob/master/The-Vector-Packet-Processor/Architectures-and-Operating-Systems.md)--Completed
  - [性能](https://github.com/penybai/vpp-docs/blob/master/The-Vector-Packet-Processor/Performance.md)--Completed
* [入门指南](https://github.com/penybai/vpp-docs/blob/master/Getting-Started/Getting-Started.md)--Completed
  - [下载和安装VPP](https://github.com/penybai/vpp-docs/blob/master/Getting-Started/Downloading-and-Installing-VPP/Downloading-and-Installing-VPP.md)
  - [VPP进阶教程](https://github.com/penybai/vpp-docs/blob/master/Getting-Started/Porgressive-VPP-Tutorial/Porgressive-VPP-Tutorial.md)
  - [用户文档](https://github.com/penybai/vpp-docs/blob/master/Getting-Started/For-Users/For-Users.md)--Completed
  - [开发者文档](https://github.com/penybai/vpp-docs/blob/master/Getting-Started/For-Developers/For-Developers.md)--Completed
  - [编写文档](https://github.com/penybai/vpp-docs/blob/master/Getting-Started/Writting-Documents/Writting-Documents.md)
* [VPP Wiki、Doxygen和其他链接](https://github.com/penybai/vpp-docs/blob/master/VPP-Wiki-Doxygen-and-Other-Links/VPP-Wiki-Doxygen-and-Other-Links.md)--Completed
  - [FD.io主站](https://github.com/penybai/vpp-docs/blob/master/VPP-Wiki-Doxygen-and-Other-Links/VPP-Wiki-Doxygen-and-Other-Links.md#FD.io主站)--Completed
  - [VPP Wiki](https://github.com/penybai/vpp-docs/blob/master/VPP-Wiki-Doxygen-and-Other-Links/VPP-Wiki-Doxygen-and-Other-Links.md#VPP-Wiki)--Completed
  - [源码文档(Doxygen)](https://github.com/penybai/vpp-docs/blob/master/VPP-Wiki-Doxygen-and-Other-Links/VPP-Wiki-Doxygen-and-Other-Links.md#源码文档(Doxygen))--Completed
* [使用案例](https://github.com/penybai/vpp-docs/blob/master/Use-Cases.md)--Completed
  - [VPP和容器](https://github.com/penybai/vpp-docs/blob/master/VPP-with-Containers.md)
  - [VPP和Iperf3、Trex](https://github.com/penybai/vpp-docs/blob/master/VPP-with-Iperf3-and-Trex.md)
  - [VPP和虚拟机](https://github.com/penybai/vpp-docs/blob/master/FD_io-VPP-with-Virutal-Machines.md)
  - [VPP和VMware/Vmxnet3](https://github.com/penybai/vpp-docs/blob/master/VPP-with-WMware-Vmxnet3.md)
  - [VPP和访问控制列表(ACL)](https://github.com/penybai/vpp-docs/blob/master/Use-Cases/Access-Control-Lists-with-FD.io-VPP/Access-Control-Lists-with-FD.io-VPP.md)--Completed
  - [VPP在云上](https://github.com/penybai/vpp-docs/blob/master/VPP-inside-the-Cloud.md)
  - [VPP作为家庭网关](https://github.com/penybai/vpp-docs/blob/master/Use-Cases/Using-VPP-as-a-Home-Gateway/Using-VPP-as-a-Home-Gateway.md)--Completed
  - [VPP和思科网络插件(Contiv)](https://github.com/penybai/vpp-docs/blob/master/Contiv-VPP.md)
  - [网络模拟器插件](https://github.com/penybai/vpp-docs/blob/master/Use-Cases/Network-Simulator-Plugin/Network-Simulator-Plugin.md)--Completed
* [故障排查](https://github.com/penybai/vpp-docs/blob/master/Troubleshooting/Troubleshooting.md)--Completed
  - [如何提交问题](https://github.com/penybai/vpp-docs/blob/master/Troubleshooting/How-to-Report-an-Issue/How-to-Report-an-Issue.md)--Completed
  - [CPU负载/使用率](https://github.com/penybai/vpp-docs/blob/master/Troubleshooting/CPU-Load-Usage/CPU-Load-Usage.md)--Completed
* [参考](https://github.com/penybai/vpp-docs/blob/master/Reference/Reference.md)--Completed
  - [有用的调试CLI](https://github.com/penybai/vpp-docs/blob/master/Reference/Useful-Debug-CLI/Useful-Debug-CLI.md)--Completed
  - [VM与Vagrant](https://github.com/penybai/vpp-docs/blob/master/Reference/VM's-with-Vagrant/VM's-with-Vagrant.md)--Completed
  - [阅读文档](https://github.com/penybai/vpp-docs/blob/master/Reference/Read-The-Docs/Read-The-Docs.md)--Completed
  - [Github仓库](https://github.com/penybai/vpp-docs/blob/master/Reference/Github-Repository/Github-Repository.md)--Completed
* [相关项目](https://github.com/penybai/vpp-docs/blob/master/Related-Projects/Related-Projects.md)--Completed
  - [CSIT](https://github.com/penybai/vpp-docs/blob/master/Related-Projects/Related-Projects.md#CSIT)--Completed
  - [TRex](https://github.com/penybai/vpp-docs/blob/master/Related-Projects/Related-Projects.md#TRex)--Completed
  - [UDPI](https://github.com/penybai/vpp-docs/blob/master/Related-Projects/Related-Projects.md#UDPI)--Completed
  - [Sweetcomb](https://github.com/penybai/vpp-docs/blob/master/Related-Projects/Related-Projects.md#Sweetcomb)--Completed
  - [Honeycomb](https://github.com/penybai/vpp-docs/blob/master/Related-Projects/Related-Projects.md#Honeycomb)--Completed
  - [GoVPP](https://github.com/penybai/vpp-docs/blob/master/Related-Projects/Related-Projects.md#GoVPP)--Completed
  - [P4VPP](https://github.com/penybai/vpp-docs/blob/master/Related-Projects/Related-Projects.md#P4VPP)--Completed
  - [ODP4VPP](https://github.com/penybai/vpp-docs/blob/master/Related-Projects/Related-Projects.md#ODP4VPP)--Completed
  - [JVPP](https://github.com/penybai/vpp-docs/blob/master/Related-Projects/Related-Projects.md#JVPP)--Completed
  - [HC2VPP](https://github.com/penybai/vpp-docs/blob/master/Related-Projects/Related-Projects.md#HC2VPP)--Completed
  - [VPP Sandbox](https://github.com/penybai/vpp-docs/blob/master/Related-Projects/Related-Projects.md#VPP-Sandbox)--Completed
  - [NSH SFC](https://github.com/penybai/vpp-docs/blob/master/Related-Projects/Related-Projects.md#NSH-SFC)--Completed
  - [ONE](https://github.com/penybai/vpp-docs/blob/master/Related-Projects/Related-Projects.md#ONE)--Completed
  - [TLDK](https://github.com/penybai/vpp-docs/blob/master/Related-Projects/Related-Projects.md#TLDK)--Completed
  - [Deb DPDK](https://github.com/penybai/vpp-docs/blob/master/Related-Projects/Related-Projects.md#Deb-DPDK)--Completed
  - [Rpm DPDK](https://github.com/penybai/vpp-docs/blob/master/Related-Projects/Related-Projects.md#Rpm-DPDK)--Completed
  - [Ci-management](https://github.com/penybai/vpp-docs/blob/master/Related-Projects/Related-Projects.md#Ci-management)--Completed
  - [Puppet-fdio](https://github.com/penybai/vpp-docs/blob/master/Related-Projects/Related-Projects.md#Puppet-fdio)--Completed
  - [Cicn](https://github.com/penybai/vpp-docs/blob/master/Related-Projects/Related-Projects.md#Cicn)--Completed
  - [Pma Tools](https://github.com/penybai/vpp-docs/blob/master/Related-Projects/Related-Projects.md#Pma-Tools)--Completed
  - [DMM](https://github.com/penybai/vpp-docs/blob/master/Related-Projects/Related-Projects.md#DMM)--Completed
  - [HICN](https://github.com/penybai/vpp-docs/blob/master/Related-Projects/Related-Projects.md#HICN)--Completed
* [关于](https://github.com/penybai/vpp-docs/blob/master/About/About.md)--Completed