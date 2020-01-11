## QUIC主机栈
quic插件提供了[IETF QUIC协议实]()现。它基于[quicly]()库。

此插件将QUIC协议添加到VPP的主机栈中。因此，QUIC可在VPP内部应用程序和外部应用程序中使用。

**成熟度**

* 此插件正在开发中：它应该可以正常工作，但尚未经过全面测试，因此不能在生产中使用。
* 目前仅支持双向流。

### 入门
* 常见的示例是将两个vpp实例互连在一起
* 确保您的vpp配置文件包含```session { evt_qs_memfd_seg }```
* 然后在调试CLI(vppctl)中运行```session enable```

该插件可以在以下情况下进行测试。

#### 内部客户端
该应用程序是在调试cli上运行的简单命令，用于通过调试cli（vppctl）在QUIC上测试连接性和吞吐量。它不能反映实际情况，主要用于内部测试。

* 在第一个实例上运行```test echo server uri quic://1.1.1.1/1234```
* 然后在第二个实例上运行```test echo client uri quic://20.20.1.1/1```
* 内部客户端的源代码位于```src/plugins/hs_apps/echo_client.c```中

#### 外部客户端
此设置反映了使用vpp创建快速客户端/服务器的应用程序开发人员的用例。该应用程序是一个外部二进制文件，通过其二进制API连接到VPP。

设置两个vpp互联后，可以将quic_echo二进制文件附加到每个vpp。

* 二进制文件在```./build-root/build-vpp[_debug]-native/vpp/bin/quic_echo```中
* 要运行客户端和服务器，请使用```quic_echo socket-name /vpp.sock client|server uri quic://1.1.1.1/1234```
* 有几个选项可用于自定义发送的数据量，线程数，日志记录和计时。

与```nclient 2/4```一起运行时，此应用程序的行为是两个首先与给定对等方建立2个连接，一旦一切都打开，就开始打开4个quic流，并传输数据。流程如下。

![](https://github.com/penybai/vpp-docs/blob/master/images/quic_plugin_echo_flow.png)

这样就可以在整个安装和拆卸阶段或者特定阶段计时来评估协议性能。

外部客户端的源代码位于```src/plugins/hs_apps/sapi/quic_echo.c```中

#### VCL客户端
主机栈公开了一个简化的API调用，称为VCL（阻止posix之类的调用），该API由支持QUIC，TCP和UDP的示例客户端和服务器实现使用。

*这些二进制文件可以在```./build-root/build-vpp[_debug]-native/vpp/bin/```中找到
* 创建VCL配置文件```echo“ vcl { api-socket-name /vpp.sock }” | tee /tmp/vcl.conf]```
* 对于服务器```VCL_CONFIG = /tmp/vcl.conf; vcl_test_server -p QUIC 1234“```
* 对于客户端```VCL_CONFIG = /tmp/vcl.conf; vcl_test_client -p QUIC 1.1.1.1 1234”```

VCL客户端的源代码位于```src/plugins/hs_apps/vcl/vcl_test_client.c```中

客户端侧的基本用法如下:
```
#include <vcl/vppcom.h>
int fd = vppcom_session_create（VPPCOM_PROTO_QUIC）;
vppcom_session_tls_add_cert（/ *参数* /）;
vppcom_session_tls_add_key（/ *参数* /）;
vppcom_session_connect（fd，“ quic：//1.1.1.1/1234”）; / *建立快速连接* /
int sfd = vppcom_session_create（VPPCOM_PROTO_QUIC）;
vppcom_session_stream_connect（sfd，fd）; / *在连接上打开一个quic流* /
vppcom_session_write（sfd，buf，n）;
```

服务器端侧
```
> #include <vcl/vppcom.h>
> int lfd = vppcom_session_create（VPPCOM_PROTO_QUIC）;
> vppcom_session_tls_add_cert（/ *参数* /）;
> vppcom_session_tls_add_key（/ *参数* /）;
> vppcom_session_bind（fd，“ quic：//1.1.1.1/1234”）;
> vppcom_session_listen（fd）;
> int fd = vppcom_session_accept（lfd）; / *接受快速连接* /
> vppcom_session_is_connectable_listener（fd）; /* 是真的 */
> int sfd = vppcom_session_accept（fd）; / *接受快速流* /
> vppcom_session_is_connectable_listener（sfd）; / *为假* /
> vppcom_session_read（sfd，buf，n）;
```

### 内部机制
QUIC结构如下：
* QUIC连接和流都是常规的主机堆栈会话，通过具有其64位句柄的API公开。
* 可以使用TRANSPORT_PROTO_QUIC进行常规的connect和close调用来创建和销毁QUIC连接。
* 可以通过再次调用connect并传递新流应属于的连接的句柄来在连接中打开流。
* 可以通过常规关闭调用来关闭流。
* 可以从与QUIC连接相对应的会话中接受对等方打开的流。
* 通过在流会话中使用常规的send和recv调用可以交换数据。

#### 数据结构
Quic依赖于主机堆栈结构，即applications，sessions，transport_connections和app_listeners。当使用quic协议侦听端口时，外部应用程序：

* 附加到vpp并注册一个```application```
* 它创建一个```app_listener```和一个```quic_listen_session```。
* ```quic_listen_session```依赖于```transport_connection（lctx）```访问```udp_listen_session```来接收数据包。
* 根据连接请求，我们创建相同的数据结构（```quic_session，qctx，udp_session```），并将句柄传递给accept回调中的```quic_session```，以确认已建立quic连接。连接两端的对等方的所有其他UDP数据报将通过```udp_session```交换。
* 接收到Stream打开请求后，我们创建```stream_session```及其传输```sctx```，并将将```stream_session```的句柄传递回应用程序。 这里没有任何UDP数据结构，因为所有数据报均已绑定到连接。

这些结构链接成如下：

![](https://github.com/penybai/vpp-docs/blob/master/images/quic_plugin_datastructures.png)