## 入门指南
入门指南由几个不同的部分组成。“用户”部分描述了VPP的基本安装和配置（手动或使用config实用程序），另一部分是“开发者”的安装，其中包含提供在开发环境中使用的工具的其他代码。

本节包括以下内容：
* 介绍如何在不同的OS平台（Ubuntu，Centos，openSUSE）上手动安装VPP Binaries，以及如何配置和使用VPP。
* 介绍在基本安装和开发人员安装中使用的不同类型的VPP软件包。
* VPP教程是学习VPP基础知识的好方法。

用户部分介绍配置操作：
* 如何手动配置和运行VPP。
* 如何使用配置实用程序进行安装，然后配置VPP。

开发者部分涵盖以下领域：
* 编译VPP
* 描述四个VPP层的组件
* 如何创建，添加，启用/禁用功能
* 讨论有界索引可扩展散列（bihash）的不同方面

编写VPP文档部分涵盖以下主题：
* 如何建立VPP文件
* 如何将您的更改推送到VPP文档存储库
* 标识与reStructuredText相关的不同样式
* 标识与Markdown相关的不同样式

* [下载和安装VPP]()
  - [在Ubuntu上安装]()
  - [在Centos上安装]()
  - [在openSUSE上安装]()
  - [安装包描述]()
* [VPP进阶教程](https://github.com/penybai/vpp-docs/blob/master/Getting-Started/Progressive-VPP-Tutorial/Progressive-VPP-Tutorial.md)
  - [设置你的环境]()
  - [运行VPP]()
  - [创建一个接口]()
  - [使用追踪命令]()
  - [连接两个FD.io VPP实例]()
  - [路由](https://github.com/penybai/vpp-docs/blob/master/Getting-Started/Progressive-VPP-Tutorial/Routing/Routing.md)
  - [交换](https://github.com/penybai/vpp-docs/blob/master/Getting-Started/Progressive-VPP-Tutorial/Switching/Switching.md)
* [用户文档]()
  - [配置VPP]()
  - [运行VPP]()
* [开发者文档]()
  - [编译VPP]()
  - [运行VPP]()
  - [GDB例子]()
  - [新增一个插件]()
  - [获得一个补丁检查]()
  - [软件架构]()
  - [VPPINFRA(基础设施)]()
  - [VLIB(矢量处理库)]()
  - [插件]()
  - [VNET(VPP网络协议栈)]()
  - [特性弧]()
  - [Buffer元数据]()
  - [Buffer元数据扩展]()
  - [多体系结构支持]()
  - [bihash(Bounded-index Extensible Hashing)]()
  - [VPP API模块]()
  - [二进制API支持]()
  - [编译系统]()
  - [事件日志(Event-Logger)]()
  - [G2图形化事件浏览器]()
  - [FIB2.0分层协议](https://github.com/penybai/vpp-docs/blob/master/Getting-Started/For-Developers/FIB-2_0-Hierarchical-Protocol-Independent/FIB-2_0-Hierarchical-Protocol-Independent.md)
  - [如何给Wireshark编译一个VPP调度跟踪器]()
  - [未决数据包(Punting Packets)](https://github.com/penybai/vpp-docs/blob/master/Getting-Started/For-Developers/Punting-Packets/Punting-Packets.md)
  - [QUIC主机协议栈]()
  - [MacOS跨平台编译]()
* [编写文档](https://github.com/penybai/vpp-docs/blob/master/Writting-Documents.md)
  - [创建VPP文档]()
  - [重组文本样式指南]()
  - [Markdown样式指南]()