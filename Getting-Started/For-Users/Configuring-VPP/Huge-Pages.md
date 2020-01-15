## 大页

VPP运行期间需要大页（hugepages），以管理大容量的内存。在VPP安装过程中，VPP将覆盖现有的大页面设置。默认情况下，VPP将系统上的大页面数设置为1024个2M大页。 这个大页数是系统上的，不只由VPP使用。

安装VPP后，将以下配置文件将复制到系统中。设置的大页在VPP安装和系统重启过程中应用，要设置大页，请执行以下命令：

```
$ cat /etc/sysctl.d/80-vpp.conf
# Number of 2MB hugepages desired
vm.nr_hugepages=1024

# Must be greater than or equal to (2 * vm.nr_hugepages).
vm.max_map_count=3096

# All groups allowed to access hugepages
vm.hugetlb_shm_group=0

# Shared Memory Max must be greater or equal to the total size of hugepages.
# For 2MB pages, TotalHugepageSize = vm.nr_hugepages * 2 * 1024 * 1024
# If the existing kernel.shmmax setting  (cat /sys/proc/kernel/shmmax)
# is greater than the calculated TotalHugepageSize then set this parameter
# to current shmmax value.
kernel.shmmax=2147483648
```

根据系统的使用方式，可以更新此配置文件以调整系统上保留的大页数。以下是一些可能的设置示例。

对于工作负荷最少的小型VM：
```
vm.nr_hugepages=512
vm.max_map_count=2048
kernel.shmmax=1073741824
```

对于大型的，运行多个VM的系统，每个VM都需要设置自己的大页数：
```
vm.nr_hugepages=32768
vm.max_map_count=66560
kernel.shmmax=68719476736
```

> 注意：如果VPP在虚拟机（VM）中运行，则该VM必须具有大页支持。安装VPP后，它将尝试覆盖现有的大页设置。如果VM没有大页支持，则安装将失败，但是该失败可能不会引起注意。重启VM后，在系统启动时，将重新应用“vm.nr_hugepages”，并且将失败，并且VM将中止内核引导，从而锁定VM。为了避免这种情况，请确保VM具有足够的大页支持。