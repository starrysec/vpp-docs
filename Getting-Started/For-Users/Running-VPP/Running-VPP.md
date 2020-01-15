## 运行VPP

### vpp用户组

安装VPP后，将创建一个新的用户组“vpp”。为避免以root用户身份运行VPP CLI（vppctl），请将需要与VPP进行交互的所有现有用户添加到新组中：
```
$ sudo usermod -a -G vpp user1
```

更新您的当前会话以使组更改生效：
```
$ newgrp vpp
```

### VPP Systemd文件-vpp.service

安装VPP后，还将安装systemd服务文件。此文件vpp.service（Ubuntu：/lib/systemd/system/vpp.service和CentOS：/usr/lib/systemd/system/vpp.service）控制如何将VPP作为服务运行。例如，是否在发生故障时重新启动，如果有，则延迟多少时间。另外，应加载哪个UIO驱动程序以及“startup.conf”文件的位置。

```
$ cat /usr/lib/systemd/system/vpp.service
[Unit]
Description=Vector Packet Processing Process
After=syslog.target network.target auditd.service

[Service]
ExecStartPre=-/bin/rm -f /dev/shm/db /dev/shm/global_vm /dev/shm/vpe-api
ExecStartPre=-/sbin/modprobe uio_pci_generic
ExecStart=/usr/bin/vpp -c /etc/vpp/startup.conf
Type=simple
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

> 注意：某些旧版本的“uio_pci_generic”驱动程序无法正确绑定所有受支持的NIC，因此需要安装从DPDK编译“igb_uio”驱动程序。此服务文件控制在引导时加载哪个驱动程序。“startup.conf”文件控制vpp使用哪个驱动程序。