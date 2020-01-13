## 获得一个补丁审核

本节主要描述如何获得vpp代码审核和合并。

### 设置

如果您没有Linux Foundation ID，请在[此处创建一个]()。

使用您的Linux Foundation ID凭据，通过gerrit.fd.io登录Gerrit Code Review。

安装git-review，这是“ Git / Gerrit提交更改或获取现有更改的命令行工具”。

如果您使用的是Ubuntu，请安装钥匙串：
```
$ sudo apt-get install keychain
```

#### ssh keys
要获取FD.io VPP文档，应使用ssh克隆VPP仓库。如上所述，您应该登录Gerrit Code Review。

使用以下命令创建您的公共和私有ssh密钥：
```
$ ssh-keygen -t rsa
$ keychain
$ cat ~/.ssh/id_rsa.pub
```

复制上述cat命令输出的公共密钥（id_rsa.pub）的所有内容。然后转到“SSH公钥设置”页面，单击“添加密钥...”，粘贴公钥，最后单击“添加”。

### 通过SSH克隆

克隆仓库：
```
$ git clone ssh://gerrit.fd.io:29418/vpp
$ cd vpp
```

仅当系统上的用户名与您的Gerrit用户名匹配时，此选项才起作用。

否则，克隆为：

```
$ git clone ssh://<YOUR_GERRIT_USERNAME>@gerrit.fd.io:29418/vpp
$ cd vpp
```

尝试克隆仓库时，Git会提示您询问是否要将服务器主机密钥添加到已知主机列表中。输入是，然后按Enter键。

### GIT审核

VPP文档使用gerrit服务器和git review来提交和获取补丁。

#### 新补丁

使用新补丁时，请使用以下命令来审核您的补丁。

通过使用以下命令，确保已修改了正确的文件：
```
$ git status
$ git diff
```

然后添加并提交补丁。您可能要在提交注释中添加标签。例如，对于仅包含补丁程序的文档，应添加标签docs:。
```
$ git add <filename>
$ git commit -s -m "<*TAG*>: <*COMMIT_MESSAGE*>"
$ git review
```

如果您正在创建的是草稿，这意味着您尚不希望对更改进行审核，请执行以下操作：
```
$ git review -D
```

提交审核后，重置HEAD指向的地方：
```
$ git reset --hard origin/master
```

#### 存在的补丁

#### 补丁冲突解决