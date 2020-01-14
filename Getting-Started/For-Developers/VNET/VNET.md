## VNET(VPP网络协议栈)
与VPP网络协议栈层关联的文件位于./src/vnet文件夹中。网络协议栈层基本上是其他层中代码的实例化。该层具有vnet库，该库提供矢量化的第2层和3个网络图节点，数据包生成器和数据包跟踪器。

在构建数据包处理应用程序方面，vnet提供了独立于平台的子图，一个子图连接了两个设备驱动程序节点。

典型的RX连接包括“ethernet-input”（完整的软件classification，提供ipv4-input，ipv6-input，arp-input等）和“ ipv4-input-no-checksum”(如果硬件可以classification，请执行ipv4头校验和)。

### 有效的图调度功能编码
在过去的15年中，出现了多种编码方式：单/双/四循环编码模型（带有变体）和完全流水线的编码模型。

### 单/双循环
单/双/四循环模型变体可以方便地解决事先不知道要处理的项目数量的问题：典型的硬件RX-ring处理。当给定节点不需要覆盖一组复杂的相关读取时，这种编码模式也非常有效。

这是一个四/单循环，可以利用avx512 SIMD向量单元(vector units)将缓冲区索引转换为缓冲区指针：

```
static uword
simulated_ethernet_interface_tx (vlib_main_t * vm,
                vlib_node_runtime_t *
                node, vlib_frame_t * frame)
{
    u32 n_left_from, *from;
    u32 next_index = 0;
    u32 n_bytes;
    u32 thread_index = vm->thread_index;
    vnet_main_t *vnm = vnet_get_main ();
    vnet_interface_main_t *im = &vnm->interface_main;
    vlib_buffer_t *bufs[VLIB_FRAME_SIZE], **b;
    u16 nexts[VLIB_FRAME_SIZE], *next;

    n_left_from = frame->n_vectors;
    from = vlib_frame_vector_args (frame);

    /*
    * Convert up to VLIB_FRAME_SIZE indices in "from" to
    * buffer pointers in bufs[]
    */
    vlib_get_buffers (vm, from, bufs, n_left_from);
    b = bufs;
    next = nexts;

    /*
    * While we have at least 4 vector elements (pkts) to process..
    */
    while (n_left_from >= 4)
    {
        /* Prefetch next quad-loop iteration. */
        if (PREDICT_TRUE (n_left_from >= 8))
    {
        vlib_prefetch_buffer_header (b[4], STORE);
        vlib_prefetch_buffer_header (b[5], STORE);
        vlib_prefetch_buffer_header (b[6], STORE);
        vlib_prefetch_buffer_header (b[7], STORE);
        }

        /*
        * $$$ Process 4x packets right here...
        * set next[0..3] to send the packets where they need to go
        */

        do_something_to (b[0]);
        do_something_to (b[1]);
        do_something_to (b[2]);
        do_something_to (b[3]);

        /* Process the next 0..4 packets */
    b += 4;
    next += 4;
    n_left_from -= 4;
}
    /*
    * Clean up 0...3 remaining packets at the end of the incoming frame
    */
    while (n_left_from > 0)
    {
        /*
        * $$$ Process one packet right here...
        * set next[0..3] to send the packets where they need to go
        */
        do_something_to (b[0]);

        /* Process the next packet */
        b += 1;
        next += 1;
        n_left_from -= 1;
    }

    /*
    * Send the packets along their respective next-node graph arcs
    * Considerable locality of reference is expected, most if not all
    * packets in the inbound vector will traverse the same next-node
    * arc
    */
    vlib_buffer_enqueue_to_next (vm, node, from, nexts, frame->n_vectors);

    return frame->n_vectors;
}
```

给定要实现的数据包处理任务，需要寻找周围的类似任务，并考虑使用相同的编码模式。在性能优化过程中，多次重新编码给定的图节点分配函数并不罕见。

### 从头开始创建数据包
有时，有必要从头开始创建数据包并将其发送。诸如发送keepalive或主动打开连接之类的任务浮现在脑海。这并不困难，但是需要准确的缓冲区元数据设置。

#### 申请缓冲区
使用vlib_buffer_alloc，它分配一组缓冲区索引。对于性能低下的应用程序，可以一次分配一个缓冲区。请注意，vlib_buffer_alloc（…）不会初始化缓冲区元数据。见下文。

在高性能情况下，分配一个缓冲区索引向量，并将其从向量末尾分发出去；在分配缓冲区索引时减少_vec_len（..）。有关示例，请参见tcp_alloc_tx_buffers（…）和tcp_get_free_buffer_index（…）。

#### 缓冲区初始化示例
下面的示例只显示了**要点**，但不是盲目的删减。
```
u32 bi0;
vlib_buffer_t *b0;
ip4_header_t *ip;
udp_header_t *udp;

/* Allocate a buffer */
if (vlib_buffer_alloc (vm, &bi0, 1) != 1)
return -1;

b0 = vlib_get_buffer (vm, bi0);

/* Initialize the buffer */
VLIB_BUFFER_TRACE_TRAJECTORY_INIT (b0);

/* At this point b0->current_data = 0, b0->current_length = 0 */

/*
* Copy data into the buffer. This example ASSUMES that data will fit
* in a single buffer, and is e.g. an ip4 packet.
*/
if (have_packet_rewrite)
    {
    clib_memcpy (b0->data, data, vec_len (data));
    b0->current_length = vec_len (data);
    }
else
    {
    /* OR, build a udp-ip packet (for example) */
    ip = vlib_buffer_get_current (b0);
    udp = (udp_header_t *) (ip + 1);
    data_dst = (u8 *) (udp + 1);

    ip->ip_version_and_header_length = 0x45;
    ip->ttl = 254;
    ip->protocol = IP_PROTOCOL_UDP;
    ip->length = clib_host_to_net_u16 (sizeof (*ip) + sizeof (*udp) +
                vec_len(udp_data));
    ip->src_address.as_u32 = src_address->as_u32;
    ip->dst_address.as_u32 = dst_address->as_u32;
    udp->src_port = clib_host_to_net_u16 (src_port);
    udp->dst_port = clib_host_to_net_u16 (dst_port);
    udp->length = clib_host_to_net_u16 (vec_len (udp_data));
    clib_memcpy (data_dst, udp_data, vec_len(udp_data));

    if (compute_udp_checksum)
        {
        /* RFC 7011 section 10.3.2. */
        udp->checksum = ip4_tcp_udp_compute_checksum (vm, b0, ip);
        if (udp->checksum == 0)
            udp->checksum = 0xffff;
    }
    b0->current_length = vec_len (sizeof (*ip) + sizeof (*udp) +
                                vec_len (udp_data));

}
b0->flags |= (VLIB_BUFFER_TOTAL_LENGTH_VALID;

/* sw_if_index 0 is the "local" interface, which always exists */
vnet_buffer (b0)->sw_if_index[VLIB_RX] = 0;

/* Use the default FIB index for tx lookup. Set non-zero to use another fib */
vnet_buffer (b0)->sw_if_index[VLIB_TX] = 0;
```
如果您的场景需要大数据包传输，请使用vlib_buffer_chain_append_data_with_alloc（…）创建必需的缓冲区链。

#### 入队查找和传输
发送一组数据包的最简单方法是使用vlib_get_frame_to_node（…）将新帧分配到ip4_lookup_node或ip6_lookup_node，添加构造的缓冲区索引，然后使用vlib_put_frame_to_node（…）分发帧。
```
    vlib_frame_t *f;
    f = vlib_get_frame_to_node (vm, ip4_lookup_node.index);
    f->n_vectors = vec_len(buffer_indices_to_send);
    to_next = vlib_frame_vector_args (f);

    for (i = 0; i < vec_len (buffer_indices_to_send); i++)
      to_next[i] = buffer_indices_to_send[i];

    vlib_put_frame_to_node (vm, ip4_lookup_node_index, f);
```
分配和调度单个分组帧效率低下，除非您需要每秒发送一个数据包，但是这种情况通常不会出现在for循环中。

### 封包跟踪器
Vlib包括一个框架元素[packet]跟踪工具，以及一个简单的调试CLI界面。cli很简单：“trace add input-node-name count”开始捕获数据包用以跟踪。

要在运行dpdk插件的典型x86_64系统上跟踪100个数据包：“trace add dpdk-input 100”。使用数据包生成器时：“trace add pg-input 100”

显示数据包跟踪：“show trace”

每个图节点都有机会捕获自己的跟踪数据。这样做总是一个好主意。跟踪捕获API很简单。

数据包捕获API对二进制数据进行快照，以最大程度地减少捕获时的处理。每个参与的图节点初始化都提供了用户自定义的vppinfra格式化功能，以在VLIB“show trace”命令时漂亮地打印数据。

将VLIB节点注册“.format_trace”成员设置为图节点格式功能的名称。

这是一个简单的示例：
```
    u8 * my_node_format_trace (u8 * s, va_list * args)
    {
        vlib_main_t * vm = va_arg (*args, vlib_main_t *);
        vlib_node_t * node = va_arg (*args, vlib_node_t *);
        my_node_trace_t * t = va_arg (*args, my_trace_t *);

        s = format (s, "My trace data was: %d", t-><whatever>);

        return s;
    }
```

跟踪框架将每个节点的格式功能传递给数据包经过时捕获的数据。格式化功能可根据需要漂亮地打印数据。

### 图调度器Pcap跟踪
vpp图调度器知道如何在调度时捕获pcap格式的数据包向量。pcap捕获如下：
```
    VPP graph dispatch trace record description:

        0                   1                   2                   3
        0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       | Major Version | Minor Version | NStrings      | ProtoHint     |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       | Buffer index (big endian)                                     |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       + VPP graph node name ...     ...               | NULL octet    |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       | Buffer Metadata ... ...                       | NULL octet    |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       | Buffer Opaque ... ...                         | NULL octet    |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       | Buffer Opaque 2 ... ...                       | NULL octet    |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       | VPP ASCII packet trace (if NStrings > 4)      | NULL octet    |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       | Packet data (up to 16K)                                       |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

图调度记录包括版本标记(Version)，和用以指示将在记录头和数据包数据之前跟随多少以NULL结尾的字符串的Nstrings字段，以及协议信息(ProtoHint)。

缓冲区索引(Buffer index)是一个不透明的32位cookie，它使这些数据的使用者在遍历转发图(traverse the forwarding graph)时可以轻松过滤/跟踪单个数据包。

每个数据包有多个记录是正常现象，这是可以预期的。数据包遍历vpp转发图时将出现多次。这样，vpp图调度跟踪与来自终端站的常规网络数据包捕获显着不同。此属性使状态数据包分析复杂化。

将状态分析限制为来自单个vpp图节点（例如“ethernet-input”）的记录似乎可以改善这种情况。

在撰写本文时：Major Version= 1，Minor Version=0，Nstrings为4或5，消费者应对小于5或者大于5的值保持警惕，如若出现这种情况，可以尝试显示要求的字符串数，或者以错误对待。

这是当前的协议集：
```
    typedef enum
      {
        VLIB_NODE_PROTO_HINT_NONE = 0,
        VLIB_NODE_PROTO_HINT_ETHERNET,
        VLIB_NODE_PROTO_HINT_IP4,
        VLIB_NODE_PROTO_HINT_IP6,
        VLIB_NODE_PROTO_HINT_TCP,
        VLIB_NODE_PROTO_HINT_UDP,
        VLIB_NODE_N_PROTO_HINTS,
      } vlib_node_proto_hint_t;
```

例如：VLIB_NODE_PROTO_HINT_IP6表示数据包数据的第一个八位字节应为0x60，并应以ipv6数据包头开头。

这些数据的下游使用者应注意协议信息(ProtoHint)。他们必须容忍不正确的协议信息，这些信息可能会不时出现。

#### 调度Pcap跟踪调试CLI
开始一个调度跟踪，捕获10000条跟踪记录：
```
    pcap dispatch trace on max 10000 file dispatch.pcap
```

开始一个调度跟踪，包含源于dpdk-input的标准的VPP数据包跟踪:
```
    pcap dispatch trace on max 10000 file dispatch.pcap buffer-trace dpdk-input 1000
```

保存pcap跟踪信息，例如：保存到/tmp/dispatch.pcap：
```
    pcap dispatch trace off
```

#### Wireshark剖析调度pcap踪迹
我们构建了一个伴随Wireshk的剖析器（dissector）来显示这些踪迹。在撰写本文时，我们已经完成了上行（upstreamed）Wireshk剖析器。

由于wireshark/master/latest进入所有流行的Linux发行版还需要一段时间，因此请参阅“如何构建可识别vpp调度跟踪的Wireshark”页面以获取构建信息。

这是数据包剖析示例，为清楚起见，省略了一些字段。关键是，wireshark剖析器可准确显示所有vpp缓冲区元数据，以及所讨论图节点的名称。
```
    Frame 1: 2216 bytes on wire (17728 bits), 2216 bytes captured (17728 bits)
        Encapsulation type: USER 13 (58)
        [Protocols in frame: vpp:vpp-metadata:vpp-opaque:vpp-opaque2:eth:ethertype:ip:tcp:data]
    VPP Dispatch Trace
        BufferIndex: 0x00036663
    NodeName: ethernet-input
    VPP Buffer Metadata
        Metadata: flags:
        Metadata: current_data: 0, current_length: 102
        Metadata: current_config_index: 0, flow_id: 0, next_buffer: 0
        Metadata: error: 0, n_add_refs: 0, buffer_pool_index: 0
        Metadata: trace_index: 0, recycle_count: 0, len_not_first_buf: 0
        Metadata: free_list_index: 0
        Metadata:
    VPP Buffer Opaque
        Opaque: raw: 00000007 ffffffff 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
        Opaque: sw_if_index[VLIB_RX]: 7, sw_if_index[VLIB_TX]: -1
        Opaque: L2 offset 0, L3 offset 0, L4 offset 0, feature arc index 0
        Opaque: ip.adj_index[VLIB_RX]: 0, ip.adj_index[VLIB_TX]: 0
        Opaque: ip.flow_hash: 0x0, ip.save_protocol: 0x0, ip.fib_index: 0
        Opaque: ip.save_rewrite_length: 0, ip.rpf_id: 0
        Opaque: ip.icmp.type: 0 ip.icmp.code: 0, ip.icmp.data: 0x0
        Opaque: ip.reass.next_index: 0, ip.reass.estimated_mtu: 0
        Opaque: ip.reass.fragment_first: 0 ip.reass.fragment_last: 0
        Opaque: ip.reass.range_first: 0 ip.reass.range_last: 0
        Opaque: ip.reass.next_range_bi: 0x0, ip.reass.ip6_frag_hdr_offset: 0
        Opaque: mpls.ttl: 0, mpls.exp: 0, mpls.first: 0, mpls.save_rewrite_length: 0, mpls.bier.n_bytes: 0
        Opaque: l2.feature_bitmap: 00000000, l2.bd_index: 0, l2.l2_len: 0, l2.shg: 0, l2.l2fib_sn: 0, l2.bd_age: 0
        Opaque: l2.feature_bitmap_input:   none configured, L2.feature_bitmap_output:   none configured
        Opaque: l2t.next_index: 0, l2t.session_index: 0
        Opaque: l2_classify.table_index: 0, l2_classify.opaque_index: 0, l2_classify.hash: 0x0
        Opaque: policer.index: 0
        Opaque: ipsec.flags: 0x0, ipsec.sad_index: 0
        Opaque: map.mtu: 0
        Opaque: map_t.v6.saddr: 0x0, map_t.v6.daddr: 0x0, map_t.v6.frag_offset: 0, map_t.v6.l4_offset: 0
        Opaque: map_t.v6.l4_protocol: 0, map_t.checksum_offset: 0, map_t.mtu: 0
        Opaque: ip_frag.mtu: 0, ip_frag.next_index: 0, ip_frag.flags: 0x0
        Opaque: cop.current_config_index: 0
        Opaque: lisp.overlay_afi: 0
        Opaque: tcp.connection_index: 0, tcp.seq_number: 0, tcp.seq_end: 0, tcp.ack_number: 0, tcp.hdr_offset: 0, tcp.data_offset: 0
        Opaque: tcp.data_len: 0, tcp.flags: 0x0
        Opaque: sctp.connection_index: 0, sctp.sid: 0, sctp.ssn: 0, sctp.tsn: 0, sctp.hdr_offset: 0
        Opaque: sctp.data_offset: 0, sctp.data_len: 0, sctp.subconn_idx: 0, sctp.flags: 0x0
        Opaque: snat.flags: 0x0
        Opaque:
    VPP Buffer Opaque2
        Opaque2: raw: 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
        Opaque2: qos.bits: 0, qos.source: 0
        Opaque2: loop_counter: 0
        Opaque2: gbp.flags: 0, gbp.src_epg: 0
        Opaque2: pg_replay_timestamp: 0
        Opaque2:
    Ethernet II, Src: 06:d6:01:41:3b:92 (06:d6:01:41:3b:92), Dst: IntelCor_3d:f6    Transmission Control Protocol, Src Port: 22432, Dst Port: 54084, Seq: 1, Ack: 1, Len: 36
        Source Port: 22432
        Destination Port: 54084
        TCP payload (36 bytes)
    Data (36 bytes)

    0000  cf aa 8b f5 53 14 d4 c7 29 75 3e 56 63 93 9d 11   ....S...)u>Vc...
    0010  e5 f2 92 27 86 56 4c 21 ce c5 23 46 d7 eb ec 0d   ...'.VL!..#F....
    0020  a8 98 36 5a                                       ..6Z
        Data: cfaa8bf55314d4c729753e5663939d11e5f2922786564c21…
        [Length: 36]
```

只需在Wireshark中单击几次鼠标即可过滤跟踪信息到特定的缓冲区索引(buffer index)。有了这种特定的过滤，人们就可以观察数据包在转发图中的走动。 注意任何/所有元数据变化，头部校验和变化等。

在开发新的vpp图节点时，这将具有重要价值。如果新的代码将b->current_data放错了位置，那么通过查看wirehark中的调度跟踪信息将显而易见。

### pcap的rx,tx和丢包跟踪
vpp还通过“pcap trace”调试CLI命令以pcap格式支持rx，tx和丢弃（drop）数据包捕获。

此命令用于启动或停止数据包捕获，或显示数据包捕获的状态。已经实现了“pcap trace rx”，“pcap trace tx”和“pcap trace drop”。同时提供“rx”，“tx”和“drop”中的一个或多个以启用多种同时捕获类型。

这些命令具有以下可选参数：
* **rx**-跟踪收到的数据包。
* **tx**-跟踪传输的数据包。
* **drop**-跟踪丢弃的数据包。
* **max ***nnnn*****-文件大小，数据包捕获的数量。接收到数据包后，跟踪缓冲区将刷新到指示的文件，默认值为1000，仅在关闭数据包捕获后才能更新。
* **max-bytes-per-pkt ***nnnn*****-每个数据包跟踪的最大字节数，必须大于32，并且小于9000。默认值：512。
* **filter**-使用必须配置的pcap rx/tx/drop trace过滤器。使用分类过滤器pcap…配置过滤器。仅当每个接口或任何接口的测试失败时，才会执行过滤器。
* **intfc ***interface***| ***any*****-用于指定给定的接口，或使用“any”在所有接口上运行数据包捕获。如果未提供，则默认为“any”。保留了先前捕获的数据包的设置，因此可以使用“any”来重置接口设置。
* **file ***filename*****-用于指定输出文件名。该文件将放置在“ /tmp”目录中。如果文件名已经存在，文件将被覆盖。如果未提供文件名，则将根据捕获方向使用“/tmp/rx.pcap或tx.pcap”。仅当关闭pcap捕获时才能更新。
* **status**-显示当前状态和与数据包捕获关联的已配置属性。如果正在进行数据包捕获，“状态”还将返回缓冲区中当前的数据包数量。在命令行中使用“状态”请求输入的所有其他属性都将被忽略。
* **filter**-捕获与当前数据包跟踪过滤器集匹配的数据包。请参阅下一节。首先配置捕获过滤器。

### 数据包跟踪捕获过滤
“classify filter pcap || trace“调试CLI命令可构造任意一组数据包分类器表，以与“pcap rx | tx | drop”配合使用，并在每个接口或整个系统范围内使用vpp数据包跟踪器。

与分类器表中的规则匹配的数据包将被跟踪，这些表会自动排序，以便首先尝试匹配最具体的表。

人们很可能会在单张表中只配置一两个匹配项。结果，默认情况下，我们配置了8个哈希存储桶（hash buckets）和128K个匹配规则空间（match rule space）。因此可以通过指定所需的“buckets”和“memory-size”来覆盖默认值。

要构建复杂的过滤器链，请反复使用"classify filter"调试CLI命令。每个命令必须指定所需的掩码和匹配值。如果已经存在带有合适掩码的分类器表，则CLI命令将匹配规则添加到现有表中。如果不是，则CLI命令添加新表和隐含的掩码规则。

#### 配置一个简单的分类过滤器
```
    classify filter pcap mask l3 ip4 src match l3 ip4 src 192.168.1.11
    pcap trace rx max 100 filter
```

#### 在一个接口上配置一个简单的捕获过滤器
```
    classify filter GigabitEthernet3/0/0 mask l3 ip4 src match l3 ip4 src 192.168.1.11
    pcap trace rx max 100 intfc GigabitEthernet3/0/0
```
需要注意的是，每个接口的捕获过滤器始终是已应用的（applied）。

#### 清除一个接口上的捕获过滤器
```
    classify filter GigabitEthernet3/0/0 del
```

#### 配置另一个相当简单的分类过滤器
```
   classify filter pcap mask l3 ip4 src dst match l3 ip4 src 192.168.1.10 dst 192.168.2.10
   pcap trace tx max 100 filter
```

#### 配置一个vpp数据包跟踪过滤器
```
   classify filter trace mask l3 ip4 src dst match l3 ip4 src 192.168.1.10 dst 192.168.2.10
   trace add dpdk-input 100 filter
```

#### 清除当前所有分类过滤器
```
    classify filter [pcap | <interface> | trace] del
```

#### 检查分类表
```
    show classify table [verbose]
```
verbose形式将显示所有匹配规则以及命中计数器。

#### 掩码(mask)的简单语法描述
```
    l2 src dst proto tag1 tag2 ignore-tag1 ignore-tag2 cos1 cos2 dot1q dot1ad
    l3 ip4 <ip4-mask> ip6 <ip6-mask>
    <ip4-mask> version hdr_length src[/width] dst[/width]
               tos length fragment_id ttl protocol checksum
    <ip6-mask> version traffic-class flow-label src dst proto
               payload_length hop_limit protocol
    l4 tcp <tcp-mask> udp <udp_mask> src_port dst_port
    <tcp-mask> src dst  # ports
    <udp-mask> src_port dst_port
```
要构造匹配项，请在掩码语法中指定的关键字之后添加要匹配的值。例如：“…mask l3 ip4 src”->“…match l3 ip4 src 192.168.1.11”

### VPP数据包生成器
我们使用VPP数据包生成器将数据包注入转发图中。数据包生成器可以重放pcap跟踪，并以相当高的性能从整个布料中生成数据包。

VPP pg支持各种用例，从新数据平面节点(new data-plane nodes)的功能测试(functional testing)到回归测试(regression testing)再到性能调整(performance tuning)。

### PG设置脚本
PG设置脚本详细描述了流量，并利用vpp调试CLI机制。构造不包含一定数量的接口和FIB配置的pg安装脚本是相当不寻常的。

例如：
```
    loop create
    set int ip address loop0 192.168.1.1/24
    set int state loop0 up

    packet-generator new {
        name pg0
        limit 100
        rate 1e6
        size 300-300
        interface loop0
        node ethernet-input
        data { IP4: 1.2.3 -> 4.5.6
               UDP: 192.168.1.10 - 192.168.1.254 -> 192.168.2.10
               UDP: 1234 -> 2345
               incrementing 286
        }
    }
```
数据包生成器流定义包括两个主要部分：

* 流参数设置
* 封包数据
  
#### 流参数设置
在上面的示例中，让我们看一下如何设置流参数：
* **name pg0**-流的名称，在上述情况下为“pg0”
* **limit 1000**-启用流时要发送的数据包数。“limit 0”表示连续发送数据包。
* **maxframe <nnn>**-最大帧大小。方便插入不大于<nnn>的多个帧。对于检查双/四循环代码很有用。
* **rate 1e6**-数据包注入速率，在上述情况下为1 MPPS。如果未指定，则数据包生成器将尽快注入数据包。
* **size 300-300**-数据包大小范围，在上述情况下，发送300字节数据包。
* **interface loop0**-数据包就像在指定接口上收到的一样。该数据有多种使用方式：选择图形弧特性配置，选择IP FIB。配置功能，例如在loop0上行使这些功能。
* **tx-interface <name>**-数据包将在指定的接口上传输。通常仅在将数据包注入IP-rewrite后图节点中时才需要。
* **pcap <filename>**-从指示的pcap捕获文件重放数据包。“make test”充分利用了此功能：使用scapy生成数据包，将其保存在.pcap文件中，然后通过vpp pg“pcap <filename>”流定义将其注入到vpp图中。
* **worker <nn>**-使用指定的vpp worker线程为流生成数据包。vpp pg生成并注入O（10 MPPS/每核）。使用多个流定义和辅助线程来生成和注入足够的流量，以轻松地用小数据包填充40 gbit管道。

#### 数据定义
数据包生成器数据定义使用分层的实现策略。网络层是按顺序指定的，这种表示方式似乎有点违反直觉。在上面的示例中，数据定义节构造了一组L2-L4标头层，并使用递增填充模式对请求的300字节数据包进行了四舍五入。

* IP4：1.2.3-> 4.5.6-使用ip4以太网类型（0x800），src MAC地址为00:01:00:02:00:03和dst MAC地址为00:04:00:05:00:06构造一个L2（MAC）头部。Mac地址可以以xxxx.xxxx.xxxx格式或xx:xx:xx:xx:xx:xx:xx格式指定。
* UDP：192.168.1.10-192.168.1.254-> 192.168.2.10-为源地址范围为.10到.254的连续数据包构造一个递增的L3（IPv4）标头集。流中的所有数据包都具有一个恒定的目的地地址192.168.2.10。将协议字段设置为17，UDP。
* UDP：1234-> 2345-将UDP源端口和目标端口分别设置为1234和2345

递增256-最多插入256个递增数据字节。

明显的变化包括上面的“s/IP4/IP6/”，以及从IPv4到IPv6地址符号的更改。

vpp pg可以设置任何/所有IPv4头部字段，包括tos，数据包长度(packet length)，mf/df/分片ID(fragment ID)和偏移量(offset)，ttl，协议，校验和和src/dst地址。详情请参加../src/vnet/ip/ip[46]_pg.c文件。

如果所有其他方法均失败，则以十六进制指定整个数据包数据：
* hex 0xabcd…-将十六进制数据逐字复制到数据包中
重放pcap文件（“pcap <filename>”）时，请勿指定数据节。

#### 诊断"packet-generator new"解析失败
如果要将数据包注入到全新的图节点中，请记住:告诉数据包生成器调试CLI如何解析数据包数据节。

如果该节点需要L2以太网MAC头部，请指定“.unformat_buffer = unformat_ethernet_header”：
```
    /* *INDENT-OFF* */
    VLIB_REGISTER_NODE (ethernet_input_node) =
    {
      <snip>
      .unformat_buffer = unformat_ethernet_header,
      <snip>
    };
```
除此之外，可能有必要在…/src/vnet/pg/cli.c中设置断点。建议使用调试镜像。

在调试新节点时，与修改包生成器相比，直接注入以太网帧并在新节点中添加相应的vlib_buffer_advance可能要简单得多。

### 调试CLI
上面详细描述了“packet-generator new”调试CLI。
更多的调试CLI命令包括：
```
    vpp# packet-generator enable [<stream-name>]
```
启用命令流或者所有流。

```
    vpp# packet-generator disable [<stream-name>]
```
禁用命名流或者所有流。

```
    vpp# packet-generator delete <stream-name>
```
删除命名流或者所有流。

```
    vpp# packet-generator configure <stream-name> [limit <nnn>]
         [rate <f64-pps>] [size <nn>-<nn>]
```
更改流参数而不必重新创建整个流定义。请注意，重新使用“packet-generator new”命令将正确地重新创建命名流。
