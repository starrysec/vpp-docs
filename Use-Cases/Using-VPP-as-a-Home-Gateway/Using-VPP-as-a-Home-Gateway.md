## VPP作为家庭网关

在小型系统（具有适当的NIC）上运行的Vpp是一个不错的家庭网关。结果系统的性能远远超出了要求：一个TAG=vpp_debug的镜像以〜1.2的矢量大小（vector size）运行，终止一个150Mbps下行/10Mbps上行电缆调制解调器的连接。

至少安装sshd和isc-dhcp-server。如果愿意，也可以使用dnsmasq。

### 配置文件

/etc/vpp/startup.conf:

```
unix {
  nodaemon
  log /var/log/vpp/vpp.log
  full-coredump
  cli-listen /run/vpp/cli.sock
  startup-config /setup.gate
  poll-sleep-usec 100
  gid vpp
}
api-segment {
  gid vpp
}
dpdk {
     dev 0000:03:00.0
     dev 0000:14:00.0
     etc.
 }

 plugins {
       ## Disable all plugins, selectively enable specific plugins
       ## YMMV, you may wish to enable other plugins (acl, etc.)
       plugin default { disable }
       plugin dpdk_plugin.so { enable }
       plugin nat_plugin.so { enable }
       ## if you plan to use the time-based MAC filter
       plugin mactime_plugin.so { enable }
 }
```

/etc/dhcp/dhcpd.conf:

```
subnet 192.168.1.0 netmask 255.255.255.0 {
  range 192.168.1.10 192.168.1.99;
  option routers 192.168.1.1;
  option domain-name-servers 8.8.8.8;
}
```

如果决定启用vpp dns域名解析器，请在dhcp服务器配置中将192.168.1.2替换为8.8.8.8。

/etc/default/isc-dhcp-server:

```
# On which interfaces should the DHCP server (dhcpd) serve DHCP requests?
#     Separate multiple interfaces with spaces, e.g. "eth0 eth1".
INTERFACESv4="lstack"
INTERFACESv6=""
```

/etc/ssh/sshd_config:

```
# What ports, IPs and protocols we listen for
Port <REDACTED-high-number-port>
# Change to no to disable tunnelled clear text passwords
PasswordAuthentication no
```

为了您自己的舒适和安全，请不要允许密码身份验证，也不要在端口22上回答ssh请求。经验表明，22端口每小时会受到多次黑客攻击，但在随机的高编号端口上则没有（或曾经没有）。

vpp配置（/setup.gate）：

```
comment { This is the WAN interface }
set int state GigabitEthernet3/0/0 up
comment { set int mac address GigabitEthernet3/0/0 mac-to-clone-if-needed }
set dhcp client intfc GigabitEthernet3/0/0 hostname vppgate

comment { Create a BVI loopback interface}
loop create
set int l2 bridge loop0 1 bvi
set int ip address loop0 192.168.1.1/24
set int state loop0 up

comment { Add more inside interfaces as needed ... }
set int l2 bridge GigabitEthernet0/14/0 1
set int state GigabitEthernet0/14/0 up

comment { dhcp server and host-stack access }
create tap host-if-name lstack host-ip4-addr 192.168.1.2/24 host-ip4-gw 192.168.1.1
set int l2 bridge tap0 1
set int state tap0 up

comment { Configure NAT}
nat44 add interface address GigabitEthernet3/0/0
set interface nat44 in loop0 out GigabitEthernet3/0/0

comment { allow inbound ssh to the <REDACTED-high-number-port> }
nat44 add static mapping local 192.168.1.2 <REDACTED> external GigabitEthernet3/0/0 <REDACTED> tcp

comment { if you want to use the vpp DNS server, add the following }
comment { Remember to adjust the isc-dhcp-server configuration appropriately }
comment { nat44 add identity mapping external GigabitEthernet3/0/0 udp 53053  }
comment { bin dns_name_server_add_del 8.8.8.8 }
comment { bin dns_name_server_add_del 68.87.74.166 }
comment { bin dns_enable_disable }
comment { see patch below, which adds these commands }
service restart isc-dhcp-server
```

### Systemd配置

在一个典型的家庭网关用例中，vpp拥有唯一的WAN链接，并希望能够到达公共互联网。诸如更新发行版软件之类的简单事情需要使用上面创建的“lstack”接口，并配置合理的上游DNS域名解析器。

如下配置/etc/systemd/resolved.conf。

/etc/systemd/resolved.conf:

```
[Resolve]
DNS=8.8.8.8
#FallbackDNS=
#Domains=
#LLMNR=no
#MulticastDNS=no
#DNSSEC=no
#Cache=yes
#DNSStubListener=yes
```

### Netplan配置

如果要在Ubuntu 18.04上的一个家庭网关以太网端口上配置静态IP地址，则需要配置netplan，Netplan相对较新。它和网络管理器（network manager）GUI可能很奇怪。在如下所示的配置中，s /enp4s0/<您的接口>/…

/etc/netplan-01-netcfg.yaml:

```
# This file describes the network interfaces available on your system
# For more information, see netplan(5).
network:
  version: 2
  renderer: networkd
  ethernets:
    enp4s0:
      dhcp4: no
      addresses: [192.168.2.254/24]
      gateway4: 192.168.2.100
      nameservers:
        search: [my.local]
        addresses: [8.8.8.8]
```

/etc/systemd/network-10.enp4s0.network:

```
[Match]
Name=enp4s0

[Link]
RequiredForOnline=no

[Network]
ConfigureWithoutCarrier=true
Address=192.168.2.254/24
```

请注意，我们为家庭网关选择了一个IP地址，该地址位于独立的不可路由子网中。这对于安装（并可能还原）新的vpp软件很方便。

### 安装新vpp软件

如果您确定给定的一组vpp Debian软件包安装后能正常运行，则可以在通过lstack/nat路径在登录网关时安装它们。此过程有点像站在地毯上并拉动它。如果一切顺利，就会发生完美的后空翻。否则，您可能希望如上所述在保留的以太网接口上配置静态IP地址。

通过ssh到192.168.1.2安装新的vpp映像：

```
# nohup dpkg -i *.deb >/dev/null 2>&1 &
```

在几秒钟之内，入站ssh连接应该再次开始响应。如果不是，则必须调试问题。

### 测试新软件

如果您经常测试新的家庭网关软件，则在生产网关后着手测试网关可能会很方便。这种测试方法减少了家庭成员的抱怨，这是其中一项好处。

将内部网络（dhcp）子网从192.168.1.0/24更改为192.168.3.0/24，将（dhcp）通告的路由器更改为192.168.3.1，将vpp Tap接口地址重新配置到192.168.3.0/24子网中，然后您应该都准备好了。

这种情况使流量受到两次干扰：首先，从192.168.3.0/24网络到192.168.1.0/24网络。接下来，从192.168.1.0/24网络连接到公共Internet。

### 补丁

您需要将此补丁来添加到“service restart”命令：

```
diff --git a/src/vpp/vnet/main.c b/src/vpp/vnet/main.c
index 6e136e19..69189c93 100644
--- a/src/vpp/vnet/main.c
+++ b/src/vpp/vnet/main.c
@@ -18,6 +18,8 @@
 #include <vlib/unix/unix.h>
 #include <vnet/plugin/plugin.h>
 #include <vnet/ethernet/ethernet.h>
+#include <vnet/ip/ip4_packet.h>
+#include <vnet/ip/format.h>
 #include <vpp/app/version.h>
 #include <vpp/api/vpe_msg_enum.h>
 #include <limits.h>
@@ -400,6 +402,63 @@ VLIB_CLI_COMMAND (test_crash_command, static) = {

 #endif

+static clib_error_t *
+restart_isc_dhcp_server_command_fn (vlib_main_t * vm,
+                                    unformat_input_t * input,
+                                    vlib_cli_command_t * cmd)
+{
+  int rv __attribute__((unused));
+  /* Wait three seconds... */
+  vlib_process_suspend (vm, 3.0);
+
+  rv = system ("/usr/sbin/service isc-dhcp-server restart");
+
+  vlib_cli_output (vm, "Restarted the isc-dhcp-server...");
+  return 0;
+}
+
+/* *INDENT-OFF* */
+VLIB_CLI_COMMAND (restart_isc_dhcp_server_command, static) = {
+  .path = "service restart isc-dhcp-server",
+  .short_help = "restarts the isc-dhcp-server",
+  .function = restart_isc_dhcp_server_command_fn,
+};
+/* *INDENT-ON* */
+
```

### 使用基于时间的mac过滤器插件

如果您需要将某些设备的网络访问限制为特定的每日时间范围，请配置“mactime”插件。将其添加到/etc/vpp/startup.conf中已启用的插件列表中，然后在NAT“内部”接口上启用该功能：

```
bin mactime_enable_disable GigabitEthernet0/14/0
bin mactime_enable_disable GigabitEthernet0/14/1
...
```

创建所需的src-mac-address规则数据库。有4种规则条目类型：
允许静态(allow-static)-通过来自此mac地址的流量
丢弃静态(drop-static)-丢弃来自此mac地址的流量
允许范围(allow-range)-在特定时间通过来自此mac地址的流量
丢弃范围(drop-range)-在特定时间丢弃来自此mac地址的流量

这里有些例子：

```
bin mactime_add_del_range name alarm-system mac 00:de:ad:be:ef:00 allow-static
bin mactime_add_del_range name unwelcome mac 00:de:ad:be:ef:01 drop-static
bin mactime_add_del_range name not-during-business-hours mac <mac> drop-range Mon - Fri 7:59 - 18:01
bin mactime_add_del_range name monday-busines-hours mac <mac> allow-range Mon 7:59 - 18:01
```