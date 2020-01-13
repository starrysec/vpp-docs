## 添加一个插件

### 概述

本章将介绍开发者如何创建一个插件，并应用到vpp中，我们假设我们从vpp的顶层目录开始。

作为一个示例，我们将使用./extras/emacs中的make-plugin.sh工具，make-plugin.sh是由一组emacs-lisp框架构造的综合插件生成器的简单包装。

#### 创建新的插件
切换目录到./src/plugins中，运行插件生成器：
```
$ cd ./src/plugins
$ ../../extras/emacs/make-plugin.sh
<snip>
Loading /scratch/vpp-docs/extras/emacs/tunnel-c-skel.el (source)...
Loading /scratch/vpp-docs/extras/emacs/tunnel-decap-skel.el (source)...
Loading /scratch/vpp-docs/extras/emacs/tunnel-encap-skel.el (source)...
Loading /scratch/vpp-docs/extras/emacs/tunnel-h-skel.el (source)...
Loading /scratch/vpp-docs/extras/emacs/elog-4-int-skel.el (source)...
Loading /scratch/vpp-docs/extras/emacs/elog-4-int-track-skel.el (source)...
Loading /scratch/vpp-docs/extras/emacs/elog-enum-skel.el (source)...
Loading /scratch/vpp-docs/extras/emacs/elog-one-datum-skel.el (source)...
Plugin name: myplugin
Dispatch type [dual or qs]: dual
(Shell command succeeded with no output)

OK...
```

插件生成器脚本会问两个问题：插件的名称以及要使用调度类型是的两种调度类型中的哪一种。由于插件名称存储到了很多地方-文件名（filenames），typedef名称（typedef names），图弧名称（graph arc names）-值得考虑一下。

调度类型是指用于构造node.c（即形式上的数据平面节点）的编码模式。**dual**选项构造带有推测性入队的双单循环对（dual-single loop pair）。这是用于负载存储密集图节点的传统编码模式。

**qs**选项生成一个使用vlib_get_buffers（…）和vlib_buffer_enqueue_to_next（…）的四单循环对（quad-single loop pair）。这些运算符充分利用了可用的SIMD矢量单位运算。如果您以后决定将四单循环对更改为双单循环对，则非常简单。

#### 生成的文件
#### 重新编译工作目录
#### 运行VPP并做完整性检查

### 生成的文件的细节
#### CMakeLists.txt
**myplugin.h**
**myplugin.c**
**myplugin_test.c**
**node.c**
#### 插件的友好性好益处
#### 更多示例