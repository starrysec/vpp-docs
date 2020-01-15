## VPP进阶教程
了解如何使用Vagrant在单个Ubuntu 16.04虚拟机上运行FD.io VPP，这涵盖了FD.io VPP基本的使用场景。本章将使用有用的VPP命令，探讨基本操作和系统上正在运行的VPP的状态。

> 注意：这并不是“如何在生产环境下运行”的使用说明集合。

关于在Virtual Box/Vargrant环境下使用VPP的更多信息，请参考：[VM和Vargrant]()。

* [设置你的环境]()
* [运行VPP]()
* [创建一个接口]()
* [使用trace命令]()
* [连接两个VPP实例]()
* [路由](https://github.com/penybai/vpp-docs/blob/master/Getting-Started/Progressive-VPP-Tutorial/Routing/Routing.md)--Completed
  - [可以学习到的技能](https://github.com/penybai/vpp-docs/blob/master/Getting-Started/Progressive-VPP-Tutorial/Routing/Routing.md#可以学习到的技能)--Completed
  - [练习学习到的命令](https://github.com/penybai/vpp-docs/blob/master/Getting-Started/Progressive-VPP-Tutorial/Routing/Routing.md#练习学习到的命令)--Completed
  - [拓扑](https://github.com/penybai/vpp-docs/blob/master/Getting-Started/Progressive-VPP-Tutorial/Routing/Routing.md#拓扑)--Completed
  - [初始状态](https://github.com/penybai/vpp-docs/blob/master/Getting-Started/Progressive-VPP-Tutorial/Routing/Routing.md#初始状态)--Completed
  - [设置主机路由](https://github.com/penybai/vpp-docs/blob/master/Getting-Started/Progressive-VPP-Tutorial/Routing/Routing.md#设置主机路由)--Completed
  - [在vpp2上设置返回路由](https://github.com/penybai/vpp-docs/blob/master/Getting-Started/Progressive-VPP-Tutorial/Routing/Routing.md#在vpp2上设置返回路由)--Completed
  - [在主机上通过vpp1 ping vpp2](https://github.com/penybai/vpp-docs/blob/master/Getting-Started/Progressive-VPP-Tutorial/Routing/Routing.md#在主机上通过vpp1-ping-vpp2)--Completed
* [交换](https://github.com/penybai/vpp-docs/blob/master/Getting-Started/Progressive-VPP-Tutorial/Switching/Switching.md)--Completed
  - [可以学习到的技能](https://github.com/penybai/vpp-docs/blob/master/Getting-Started/Progressive-VPP-Tutorial/Switching/Switching.md#可以学习到的技能)--Completed
  - [练习学习到的命令](https://github.com/penybai/vpp-docs/blob/master/Getting-Started/Progressive-VPP-Tutorial/Switching/Switching.md#练习学习到的命令)--Completed
  - [拓扑](https://github.com/penybai/vpp-docs/blob/master/Getting-Started/Progressive-VPP-Tutorial/Switching/Switching.md#拓扑)--Completed
  - [初始状态](https://github.com/penybai/vpp-docs/blob/master/Getting-Started/Progressive-VPP-Tutorial/Switching/Switching.md#初始状态)--Completed
  - [运行FD.io VPP实例](https://github.com/penybai/vpp-docs/blob/master/Getting-Started/Progressive-VPP-Tutorial/Switching/Switching.md#运行FD.io-VPP实例)--Completed
  - [vpp1连接主机](https://github.com/penybai/vpp-docs/blob/master/Getting-Started/Progressive-VPP-Tutorial/Switching/Switching.md#vpp1连接主机)--Completed
  - [vpp1连接vpp2](https://github.com/penybai/vpp-docs/blob/master/Getting-Started/Progressive-VPP-Tutorial/Switching/Switching.md#vpp1连接vpp2)--Completed
  - [配置vpp1上的桥域](https://github.com/penybai/vpp-docs/blob/master/Getting-Started/Progressive-VPP-Tutorial/Switching/Switching.md#配置vpp1上的桥域)--Completed
  - [配置vpp2上的loopback接口](https://github.com/penybai/vpp-docs/blob/master/Getting-Started/Progressive-VPP-Tutorial/Switching/Switching.md#配置vpp2上的loopback接口)--Completed
  - [配置vpp2上的桥域](https://github.com/penybai/vpp-docs/blob/master/Getting-Started/Progressive-VPP-Tutorial/Switching/Switching.md#配置vpp2上的桥域)--Completed
  - [从主机ping vpp和从vpp ping主机](https://github.com/penybai/vpp-docs/blob/master/Getting-Started/Progressive-VPP-Tutorial/Switching/Switching.md#从主机ping-vpp和从vpp-ping主机)--Completed
  - [检查l2 fib](https://github.com/penybai/vpp-docs/blob/master/Getting-Started/Progressive-VPP-Tutorial/Switching/Switching.md#检查l2-fib)--Completed