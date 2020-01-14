## FIB规模
FIB规模的唯一限制因素是分配给FIB使用的每个堆的内存量，共有4个：
* IP4堆
* IP6堆
* 主堆
* 统计堆

## IPv4堆
IPv4堆用于分配存储IPv4前缀的数据结构所需的内存。用户创建的每个表，即;
```
$ ip table add 1
```
或默认表，包含2个ip4_fib_t对象。“非转发”ip4_fib_t包含表中的所有条目，“转发”包含在数据平面中匹配的条目。两组之间的区别是数据平面中不应匹配的条目。每个ip4_fib_t包含一个mtrie（用于在数据平面中快速查找）和一个哈希表每个前缀长度（用于在控制平面中查找）。

要查看IPv4表消耗的内存量，请使用：

```
vpp# sh ip fib mem
ipv4-VRF:0 mtrie:333056 hash:3523
ipv4-VRF:1 mtrie:333056 hash:3523
totals: mtrie:666112 hash:7046 all:673158

Mtrie Mheap Usage: total: 32.06M, used: 662.44K, free: 31.42M, trimmable: 31.09M
    free chunks 3 free fastbin blks 0
    max total allocated 32.06M
no traced allocations
```
此输出显示两个“空”（即未添加路由）表。每个mtrie使用大约150k的内存，因此每个表大约300k。最后显示IP4堆的总堆使用情况统计信息。

下边的输出是分别添加了1M，2M和4M路由：
```
vpp# sh ip fib mem
ipv4-VRF:0 mtrie:335744 hash:4695
totals: mtrie:335744 hash:4695 all:340439

Mtrie Mheap Usage: total: 1.00G, used: 335.20K, free: 1023.74M, trimmable: 1023.72M
    free chunks 3 free fastbin blks 0
    max total allocated 1.00G
no traced allocations
```

```
vpp# sh ip fib mem
ipv4-VRF:0 mtrie:5414720 hash:41177579
totals: mtrie:5414720 hash:41177579 all:46592299

Mtrie Mheap Usage: total: 1.00G, used: 46.87M, free: 977.19M, trimmable: 955.93M
    free chunks 61 free fastbin blks 0
    max total allocated 1.00G
no traced allocations
```

```
vpp# sh ip fib mem
ipv4-VRF:0 mtrie:22452608 hash:168544508
totals: mtrie:22452608 hash:168544508 all:190997116

Mtrie Mheap Usage: total: 1.00G, used: 198.37M, free: 825.69M, trimmable: 748.24M
    free chunks 219 free fastbin blks 0
    max total allocated 1.00G
no traced allocations
```
VPP以1G IPv4堆开始。

## IPv6堆
IPv6堆用于分配存储IPv6前缀的数据结构所需的内存。IPv6也具有转发和非转发条目的概念，但是对于IPv6，所有转发条目都存储在单个哈希表中（非转发也是如此）。哈希表的关键字包括IPv6表ID。

要查看IPv6表消耗的内存量，请使用：
```
vpp# sh ip6 fib mem
IPv6 Non-Forwarding Hash Table:
Hash table ip6 FIB non-fwding table
    7 active elements 7 active buckets
    1 free lists
    0 linear search buckets
    arena: base 7f2fe28bf000, next 803c0
           used 525248 b (0 Mbytes) of 33554432 b (32 Mbytes)

IPv6 Forwarding Hash Table:
Hash table ip6 FIB fwding table
    7 active elements 7 active buckets
    1 free lists
    0 linear search buckets
    arena: base 7f2fe48bf000, next 803c0
           used 525248 b (0 Mbytes) of 33554432 b (32 Mbytes)
```

当我们扩展到128k IPv6表项时:

```
vpp# sh ip6 fib mem
IPv6 Non-Forwarding Hash Table:
Hash table ip6 FIB non-fwding table
    131079 active elements 32773 active buckets
    2 free lists
       [len 1] 2 free elts
    0 linear search buckets
    arena: base 7fed7a514000, next 4805c0
           used 4720064 b (4 Mbytes) of 1073741824 b (1024 Mbytes)

IPv6 Forwarding Hash Table:
Hash table ip6 FIB fwding table
    131079 active elements 32773 active buckets
    2 free lists
       [len 1] 2 free elts
    0 linear search buckets
    arena: base 7fedba514000, next 4805c0
           used 4720064 b (4 Mbytes) of 1073741824 b (1024 Mbytes)
```

扩展到256k时：

```
vpp# sh ip6 fib mem
IPv6 Non-Forwarding Hash Table:
Hash table ip6 FIB non-fwding table
    262151 active elements 65536 active buckets
    2 free lists
       [len 1] 6 free elts
    0 linear search buckets
    arena: base 7fed7a514000, next 880840
           used 8915008 b (8 Mbytes) of 1073741824 b (1024 Mbytes)

IPv6 Forwarding Hash Table:
Hash table ip6 FIB fwding table
    262151 active elements 65536 active buckets
    2 free lists
       [len 1] 6 free elts
    0 linear search buckets
    arena: base 7fedba514000, next 880840
           used 8915008 b (8 Mbytes) of 1073741824 b (1024 Mbytes)
```

扩展到1M时：

```
vpp# sh ip6 fib mem
IPv6 Non-Forwarding Hash Table:
Hash table ip6 FIB non-fwding table
    1048583 active elements 65536 active buckets
    4 free lists
       [len 1] 65533 free elts
       [len 2] 65531 free elts
       [len 4] 9 free elts
    0 linear search buckets
    arena: base 7fed7a514000, next 3882740
           used 59254592 b (56 Mbytes) of 1073741824 b (1024 Mbytes)

IPv6 Forwarding Hash Table:
Hash table ip6 FIB fwding table
    1048583 active elements 65536 active buckets
    4 free lists
       [len 1] 65533 free elts
       [len 2] 65531 free elts
       [len 4] 9 free elts
    0 linear search buckets
    arena: base 7fedba514000, next 3882740
           used 59254592 b (56 Mbytes) of 1073741824 b (1024 Mbytes)
```

从输出可以看出，在这种情况下，IPv6堆已扩展为1GB，而100万个前缀已使用了其中的56MB。

## 主堆

主堆用于分配表示控制平面和数据平面中的FIB条目的对象（请参见[控制平面]()和[数据平面]()），例如fib_entry_t和load_balance_t。这些来自主堆，因为它们不是特定于协议的（即，它们用于表示IPv4，IPv6或MPLS条目）。

分配了1M前缀后，内存使用情况为：
```
vpp# sh fib mem
FIB memory
 Tables:
            SAFI              Number     Bytes
        IPv4 unicast             1     33619968
        IPv6 unicast             2     118502784
            MPLS                 0         0
       IPv4 multicast            1       1175
       IPv6 multicast            1      525312
 Nodes:
            Name               Size  in-use /allocated   totals
            Entry               72   1048589/ 1048589    75498408/75498408
        Entry Source            40   1048589/ 1048589    41943560/41943560
    Entry Path-Extensions       76      0   /    0       0/0
       multicast-Entry         192      6   /    6       1152/1152
          Path-list             40     18   /    18      720/720
          uRPF-list             16     14   /    14      224/224
            Path                72     22   /    22      1584/1584
     Node-list elements         20   1048602/ 1048602    20972040/20972040
       Node-list heads          8      24   /    24      192/192
```

2M时：

```
vpp# sh fib mem
FIB memory
 Tables:
            SAFI              Number     Bytes
        IPv4 unicast             1     33619968
        IPv6 unicast             2     252743040
            MPLS                 0         0
       IPv4 multicast            1       1175
       IPv6 multicast            1      525312
 Nodes:
            Name               Size  in-use /allocated   totals
            Entry               72   2097165/ 2097165    150995880/150995880
        Entry Source            40   2097165/ 2097165    83886600/83886600
    Entry Path-Extensions       76      0   /    0       0/0
       multicast-Entry         192      6   /    6       1152/1152
          Path-list             40     18   /    19      720/760
          uRPF-list             16     18   /    18      288/288
            Path                72     22   /    23      1584/1656
     Node-list elements         20   2097178/ 2097178    41943560/41943560
       Node-list heads          8      24   /    24      192/192
```

但是，情况并非如此简单。上面添加的所有1M前缀都可以通过同一个下一跳访问，因此它们使用的路径列表（和路径）是共享的。随着添加使用不同（下一跳）（多组）下一跳的前缀，所需的路径列表和路径的数量将增加。

## 状态堆

VPP收集每个路由的统计信息。对于每条路由，VPP都会收集字节和数据包计数器，以了解发送到该前缀的数据包（即该路由在数据平面中已匹配）以及通过该前缀发送的数据包（即匹配的前缀可以通过它到达-就像BGP对等体（peer）一样）。这需要在统计信息段中为每个路由分配4个计数器。

下面显示了具有1M，2M和4M路由的统计信息段的大小。
```
total: 1023.99M, used: 127.89M, free: 896.10M, trimmable: 830.94M
total: 1023.99M, used: 234.14M, free: 789.85M, trimmable: 668.15M
total: 1023.99M, used: 456.83M, free: 567.17M, trimmable: 388.91M
```
VPP以1G状态堆开始。
