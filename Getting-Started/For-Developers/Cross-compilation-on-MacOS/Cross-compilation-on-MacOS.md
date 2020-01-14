## MacOS跨平台编译

这是首次尝试在MacOS上支持VPP交叉编译以进行开发（linting，completion，compile_commands.json）

**先决条件**

* 您需要安装以下软件包:
* 您还需要安装`gnu-ident 2.2.11`，使得能够`make checkstyle`，您可以从GNU处获取。
* 您应链接到二进制文件，以使其在路径中可用，并带有其原始名称，例如：
```
ln -s $(which gsed) /usr/local/bin/sed
ln -s $(which gindent) /usr/local/bin/indent
ln -s /usr/local/Cellar/diffutils/3.7/bin/diff /usr/local/bin/diff
```

**设置**

* 创建一个交叉编译工具链
* 创建区分大小写的卷并在其中安装工具链，例如在`/Volumes/xchain`中
* 使用`$VPP_DIR/extras/scripts/cross_compile_macos.sh conf /Volumes/xchan`创建一个xchain.toolchain文件

目前，我们不支持e-build，因此不会将dpdk，rdma，quicly编译为`make build`的一部分。

要使用工具链进行编译，请执行以下操作：
```
$VPP_DIR/extras/scripts/cross_compile_macos.sh build
```
要获取compile_commands.json，请执行:
```
$ VPP_DIR/extras/scripts/cross_compile_macos.sh cc
＃>> ./build-root/build-vpp[_debug]-native/vpp/compile_commands.json
```
这样应该能够在MacOS上编译vpp。

祝好运:)