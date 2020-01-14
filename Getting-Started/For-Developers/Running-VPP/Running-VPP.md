## 运行VPP
编译VPP二进制文件后，您现在可能已编译了多个镜像。当您需要运行VPP而不安装软件包时，这些映像很有用。例如，如果要与GDB一起运行VPP。

### 不通过GDB运行
不通过GDB运行您的VPP，请使用如下命令：

运行发行版（release）镜像：
```
# make run-release
#
```

运行调试版（debug）镜像：
```
# make run
#
```

### 通过GDB运行
使用以下命令，您可以运行VPP，然后进入到GDB提示符。

运行发行版（release）镜像：
```
# make debug-release
(gdb)
```

运行调试版（debug）镜像：
```
# make debug
(gdb)
```