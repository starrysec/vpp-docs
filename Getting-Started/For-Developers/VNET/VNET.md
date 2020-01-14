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

####
####

### pcap的rx,tx和丢包跟踪
### 数据包跟踪捕获过滤
### VPP数据包生成器
### PG设置脚本
### 调试CLI