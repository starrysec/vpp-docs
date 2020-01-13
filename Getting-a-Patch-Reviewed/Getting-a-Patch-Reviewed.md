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

#### 现有补丁

下边的审核的URL中有“更改编号（change number）”.

点击单个审核后，可在URL的“https://gerrit.fd.io/r/#/c/<CHANGE_NUMBER>/”找到”更改编号“。

查看现有的补丁：
```
$ git review -d <change number>
$ git status
$ git diff
```

> 警告：如果进行了更改并执行了“git review -d <更改编号>”，则将尝试隐藏当前更改，以便工作树可以更改为您指定的审核分支。如果要确保不会丢失更改，请使用ssh克隆中显示的克隆步骤将另一个Gerrit存储库克隆到新目录中，然后在此新目录中执行“git review -d <change number>”。

修改现有的补丁，确保你修改了正确的文件，并应用补丁：
```
$ git review -d <change number>
$ git status
$ git diff

$ git add <filename>
$ git commit --amend
$ git review
```

当您完成查看或者修改一个分支，回到主分支通过输入如下命令：

```
$ git reset --hard origin/master
$ git checkout master
```

#### 补丁冲突解决

有时会出现两种不同的补丁冲突情况。将补丁上传到https://gerrit.fd.io后的某个时候，gerrit UI可能会显示补丁状态为“合并冲突”。

或者，您可能尝试通过“git review”上传新的补丁集，只是发现gerrit服务器由于上游合并冲突而不允许上传。

在两种情况下，通常都非常容易解决问题。您需要将补丁重新设置为主/最新(master/latest)版本。细节因情况而异。

以下是重新设置先前上传到Gerrit服务器的补丁的方法，该补丁现在有合并冲突。在从主/最新克隆的新工作区中，执行以下操作：

```
$ git-review -d <*Gerrit change #*>
$ git rebase origin/master
   while (conflicts)
      <fix conflicts>
      $ git rebase --continue
$ git review
```

如果出现上传失败的情况，请格外小心：在执行其他任何操作之前，请仔细保存工作！

重新安装补丁程序，然后重试。请不要从gerrit服务器重新下载[“git review -d”]补丁…：

```
$ git rebase origin/master
   while (conflicts)
      <fix conflicts>
      $ git rebase --continue
$ git review
```