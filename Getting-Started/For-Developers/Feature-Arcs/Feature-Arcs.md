## 特性弧
可以在每个接口或每个系统的基础上配置大量的vpp功能。我们建立了管理这些特性的机制，而不需要特性编码器手动构造所需要的图弧（graph arcs）。

具体来说，特性弧是包含多个图节点的有序集合。弧中的每个特性节点都是独立控制的，特性弧节点之间通常不会互相知道对方的存在。因此，将数据包传递到“下一个特性节点”非常容易。

特性弧的实现解决了数据包处理流程转向的图节点创建问题。

在特性弧的开头，需要进行一些设置工作，但前提是必须在弧上启用至少一个特性。

在每个弧的基础上，各个特性定义会创建一组排序依存关系。特性基础设施依据拓扑关系执行依赖排序，以确定实际的功能顺序。缺少依赖项将导致运行时混乱。有关示例，请参见https://gerrit.fd.io/r/#/c/12753。

如果部分排序不存在，则vpp将拒绝运行。 “ a然后b，b然后c，c然后a”形式的循环依赖是无法满足的。

### 向已经存在的特性弧上添加特性
一点都不用惊讶，我们使用典型的“宏->构造函数->声明列表”模式设置特性弧：
```
VNET_FEATURE_INIT (mactime, static) =
    {
      .arc_name = "device-input",
      .node_name = "mactime",
      .runs_before = VNET_FEATURES ("ethernet-input"),
    };  
```

这样就在“device-input”弧上添加了一个“mactime”的特性。

每帧一次，挖掘对应于“device-input”特性弧的vnet_feature_config_main_t：
```
 vnet_main_t *vnm = vnet_get_main ();
    vnet_interface_main_t *im = &vnm->interface_main;
    u8 arc = im->output_feature_arc_index;
    vnet_feature_config_main_t *fcm;

    fcm = vnet_feature_get_config_main (arc);
```

请注意，在这种情况下，我们已将所需的弧索引（由特性弧基础设施分配）存储在vnet_interface_main_t中。创建特性弧时，程序员将决定放置弧索引的位置。

对于每个数据包，设置next0以使数据包引导到它们应访问的下一个节点：
```
vnet_get_config_data (&fcm->config_main,
                      &b0->current_config_index /* value-result */, 
                      &next0, 0 /* # bytes of config data */);
```

配置数据是每个特性弧所有的，通常不使用。 请注意，重置next0将数据包转移到其他地方是正常的， 通常，出于一些原因将其丢弃：

```
    next0 = MACTIME_NEXT_DROP;
    b0->error = node->errors[DROP_CAUSE];
```

### 创建一个特性弧
通常，我是使用构造函数宏创建特性弧：
```
VNET_FEATURE_ARC_INIT (ip4_unicast, static) =
    {
      .arc_name = "ip4-unicast",
      .start_nodes = VNET_FEATURES ("ip4-input", "ip4-input-no-checksum"),
      .arc_index_ptr = &ip4_main.lookup_main.ucast_feature_arc_index,
    };  
```

在这种情况下，我们配置两个弧起始节点以处理“是否经过硬件验证的ip校验和”情况。在初始化期间，特性基础设施将存储弧索引，如下所示。

在弧节点中，执行以下操作以沿特性弧发送数据包：

```
ip_lookup_main_t *lm = &im->lookup_main;
arc = lm->ucast_feature_arc_index;
```

对于每一个数据包，初始化数据包元数据来使数据包沿着特性弧前进：

```
vnet_feature_arc_start (arc, sw_if_index0, &next, b0);
```

### 启用/禁用特性
通过简单调用vnet_feature_enable_disable来启用和禁用特性弧：
```
 vnet_feature_enable_disable ("device-input", /* arc name */
                              "mactime",      /* feature name */
           		              sw_if_index,    /* Interface sw_if_index */
                              enable_disable, /* 1 => enable */
                              0 /* (void *) feature_configuration */, 
                              0 /* feature_configuration_nbytes */);
```

不透明参数feature_configuration很少使用。

如果您希望将特性设置为事实上的系统级概念，请始终传递sw_if_index = 0，sw_if_index为0始终有效，并且对应于“local”接口。

### 相关的show命令
要显示全部特性，请使用“show features [verbose]”命令。详细形式显示弧索引，并在弧内显示特性：
```
$ vppctl show features verbose
Available feature paths
<snip>
[14] ip4-unicast:
  [ 0]: nat64-out2in-handoff
  [ 1]: nat64-out2in
  [ 2]: nat44-ed-hairpin-dst
  [ 3]: nat44-hairpin-dst
  [ 4]: ip4-dhcp-client-detect
  [ 5]: nat44-out2in-fast
  [ 6]: nat44-in2out-fast
  [ 7]: nat44-handoff-classify
  [ 8]: nat44-out2in-worker-handoff
  [ 9]: nat44-in2out-worker-handoff
  [10]: nat44-ed-classify
  [11]: nat44-ed-out2in
  [12]: nat44-ed-in2out
  [13]: nat44-det-classify
  [14]: nat44-det-out2in
  [15]: nat44-det-in2out
  [16]: nat44-classify
  [17]: nat44-out2in
  [18]: nat44-in2out
  [19]: ip4-qos-record
  [20]: ip4-vxlan-gpe-bypass
  [21]: ip4-reassembly-feature
  [22]: ip4-not-enabled
  [23]: ip4-source-and-port-range-check-rx
  [24]: ip4-flow-classify
  [25]: ip4-inacl
  [26]: ip4-source-check-via-rx
  [27]: ip4-source-check-via-any
  [28]: ip4-policer-classify
  [29]: ipsec-input-ip4
  [30]: vpath-input-ip4
  [31]: ip4-vxlan-bypass
  [32]: ip4-lookup
<snip>
```

在这里，我们了解到ip4-unicast特性弧的索引为14，且ip4-inacl是生成的部分顺序中的第25个特性。

要显示特定接口上激活的特性，请使用“show interface features”命令：

```
$ vppctl show interface GigabitEthernet3/0/0 features
Feature paths configured on GigabitEthernet3/0/0...
<snip>
ip4-unicast:
  nat44-out2in
<snip>
```

### 特性弧表
只需搜索名称字符串即可跟踪弧的定义、索引的位置等。
```
            |    Arc Name      |
            |------------------|
            | device-input     |
            | ethernet-output  |
            | interface-output |
            | ip4-drop         |
            | ip4-local        |
            | ip4-multicast    |
            | ip4-output       |
            | ip4-punt         |
            | ip4-unicast      |
            | ip6-drop         |
            | ip6-local        |
            | ip6-multicast    |
            | ip6-output       |
            | ip6-punt         |
            | ip6-unicast      |
            | mpls-input       |
            | mpls-output      |
            | nsh-output       |
```