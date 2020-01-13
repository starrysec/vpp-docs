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
下边是生成的文件，我们快速浏览以下：
```
$ cd ./myplugin
$ ls
CMakeLists.txt        myplugin.c           myplugin_periodic.c  setup.pg
myplugin_all_api_h.h  myplugin.h           myplugin_test.c
myplugin.api          myplugin_msg_enum.h  node.c
```

由于最近改进了编译系统，因此您无需其他文件即可将新插件集成到vpp构建中。只需在工作目录从现编译，就会出现新的插件。

#### 重新编译工作目录
使用下边的方法，简单得重新配置和编译你的工作目录：

```
$ cd <top-of-workspace>
$ make rebuild [or rebuild-release]
```

多谢缓存，整个重新编译过程不需要花费太多时间。

#### 运行VPP并做完整性检查
做一个快速的完整行检查，然后运行，确保“myplugin_plugin.so”和“myplugin_test_plugin.so”已经被加载：
```
$ cd <top-of-workspace>
$ make run
<snip>
load_one_plugin:189: Loaded plugin: myplugin_plugin.so (myplugin description goes here)
<snip>
load_one_vat_plugin:67: Loaded plugin: myplugin_test_plugin.so
<snip>
DBGvpp#
```

如果这个简单的检查失败，请寻求帮助。

### 生成的文件的细节

本节讨论一些生成的文件的细节，可以略过本节，稍后再返回以获取更多详细信息。

#### CMakeLists.txt

这是编译系统编译插件的秘诀，请修正版权声明：
```
# Copyright (c) <current-year> <your-organization>
```

其余的编译秘诀很简单：
```
add_vpp_plugin (myplugin
SOURCES
myplugin.c
node.c
myplugin_periodic.c
myplugin.h

MULTIARCH_SOURCES
node.c

API_FILES
myplugin.api

INSTALL_HEADERS
myplugin_all_api_h.h
myplugin_msg_enum.h

API_TEST_SOURCES
myplugin_test.c
)
```

如您所见，编译秘诀由几个文件列表组成。SOURCES是C源文件的列表。API_FILES是插件的二进制API定义文件的列表（此文件通常很多），依此类推。

MULTIARCH_SOURCES列出了认为对性能至关重要的数据平面图节点调度功能源文件。这些文件中的特定功能会被多次编译，以便它们可以利用特定于CPU的特性。稍后对此进行更多讨论。

如果添加源文件，只需将它们添加到指定的列表中即可。

**myplugin.h**

这是新插件的主要#include文件。除其他外，它定义了插件的main_t数据结构。这是添加特定问题的数据结构的正确位置。请抵制在插件中创建一组静态或[更糟糕]全局变量的诱惑。判断插件之间的名称冲突对任何人来说都不是件快乐的事。

**myplugin.c**

为了更好地描述它，myplugin.c是vpp插件的“main.c”等效项。它的工作是将插件连接到vpp二进制API消息分发器中，并将其消息添加到vpp的全局“message-name_crc”哈希表中。请参见“myplugin_init（…”）”

Vpp本身使用dlsym（…）来跟踪由VLIB_PLUGIN_REGISTER宏生成的vlib_plugin_registration_t：

```
VLIB_PLUGIN_REGISTER () =
  {
    .version = VPP_BUILD_VER,
    .description = "myplugin plugin description goes here",
  };
```

vpp仅从插件目录中加载含有这个数据结构实例的.so文件。

你可以通过命令行启用和禁用指定的插件，默认情况下，插件已加载。若要更改该行为，请在宏VLIB_PLUGIN_REGISTER中设置default_disabled:
```
VLIB_PLUGIN_REGISTER () =
  {
    .version = VPP_BUILD_VER,
    .default_disabled = 1
    .description = "myplugin plugin description goes here",
  };
```

模版生成器将图节点分发（dispatch）功能放置到“device-input”特性弧上。这可能有用也可能没有用。

```
VNET_FEATURE_INIT (myplugin, static) =
{
  .arc_name = "device-input",
  .node_name = "myplugin",
  .runs_before = VNET_FEATURES ("ethernet-input"),
};
```

如插件生成器所给出的那样，myplugin.c包含二进制API消息处理程序，用于通用的“请在这样的接口上启用我的功能”二进制API消息。如您所见，设置vpp消息API表很简单。大警告：该方案不能容忍小错误。例如：忘记添加mainp->msg_id_base会导致非常迷惑的失败。

如果您坚持谨慎地修改生成的样板-而不是尝试按照首要原则来编译代码-您将节省很多时间和精力。

**myplugin_test.c**

该文件包含二进制API消息生成代码，该代码被编译成单独的.so文件。“vpp_api_test”程序将加载这些插件，从而立即访问您的插件API，以进行外部客户端二进制API测试。

vpp本身会加载测试插件，并通过“binary-api”调试CLI使代码可用。这是在集成测试之前对二进制API进行单元测试的一种常用方法。

**node.c**

这是生成的图节点分派函数。您需要重写它来解决当前的问题。保留节点分发功能的结构将节省大量时间和麻烦。

即使是专家，也要浪费时间来重新构造循环结构，排队模式等等。只需撕下样本1x，2x，4x数据包处理代码，然后将其替换为与您要解决的问题相关的代码即可。

#### 插件的友好性好益处

在vpp VLIB_INIT_FUNCTION函数中，通常可以看到特定的init函数调用其他init函数：

```
if ((error = vlib_call_init_function (vm, some_other_init_function))
   return error;
```

如果一个插件需要在另一个插件中调用init函数，请使用vlib_call_plugin_init_function宏：

```
if ((error = vlib_call_plugin_init_function (vm, "otherpluginname", some_init_function))
   return error;
```

这允许在插件初始化函数之间进行排序。

如果您希望获得指向另一个插件中的符号的指针，请使用vlib_plugin_get_symbol（…）API：

```
void *p = vlib_get_plugin_symbol ("plugin_name", "symbol");
```

#### 更多示例
更多信息请阅读./src/plugins中的示例。