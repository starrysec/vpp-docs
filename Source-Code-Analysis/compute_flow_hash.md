## 计算flow哈希

对于以太网数据包，IPv4和IPv6默认使用5元组（smac/dmac/sip/dip/proto）计算流哈希（对于tcp协议再增加sport/dport），其他默认使用3元组（smac/dmac/etype）计算流哈希。
当然，也可以通过配置修改流哈希生成规则（flow_hash_config）。

另外还有mpls流哈希计算（文件`vnet/mpls/mpls_lookup.h`中的mpls_compute_flow_hash()函数）和bier流哈希计算（文件`vnet/bier/bier_fwd.h`中的bier_compute_flow_hash()函数）。

### L2流哈希计算

L2流哈希计算源码位于`vent/l2/l2_input.h`文件中。

```
/*
 * Compute flow hash of an ethernet packet, use 5-tuple hash if L3 packet
 * is ip4 or ip6. Otherwise hash on smac/dmac/etype.
 * The vlib buffer current pointer is expected to be at ethernet header
 * and vnet l2.l2_len is expected to be setup already.
 */
static inline u32
vnet_l2_compute_flow_hash (vlib_buffer_t * b)
{
  // 2层头
  ethernet_header_t *eh = vlib_buffer_get_current (b);
  // 3层头
  u8 *l3h = (u8 *) eh + vnet_buffer (b)->l2.l2_len;
  u16 ethertype = clib_net_to_host_u16 (*(u16 *) (l3h - 2));

  // IPv4
  if (ethertype == ETHERNET_TYPE_IP4)
    return ip4_compute_flow_hash ((ip4_header_t *) l3h, IP_FLOW_HASH_DEFAULT);
  // IPv6
  else if (ethertype == ETHERNET_TYPE_IP6)
    return ip6_compute_flow_hash ((ip6_header_t *) l3h, IP_FLOW_HASH_DEFAULT);
  // 其他，使用三元组（smac/dmac/etype）计算。
  else
    {
      u32 a, b, c;
      u32 *ap = (u32 *) & eh->dst_address[2];
      u32 *bp = (u32 *) & eh->src_address[2];
      a = *ap;
      b = *bp;
      c = ethertype;
      hash_v3_mix32 (a, b, c);
      hash_v3_finalize32 (a, b, c);
      return c;
    }
}
```

### L3流哈希计算

#### IPv4流哈希计算

IPv4流哈希计算源码位`vnet/ip/ip4.h`于文件中。

TCP协议，五元组基础上增加sport/dport。
UDP协议，由于IP分片存在，在没分片重组前无法获取端口，使用0/0。

```
/* Compute flow hash.  We'll use it to select which adjacency to use for this
   flow.  And other things. */
always_inline u32
ip4_compute_flow_hash (const ip4_header_t * ip,
		       flow_hash_config_t flow_hash_config)
{
  tcp_header_t *tcp = (void *) (ip + 1);
  u32 a, b, c, t1, t2;
  uword is_tcp_udp = (ip->protocol == IP_PROTOCOL_TCP
		      || ip->protocol == IP_PROTOCOL_UDP);

  t1 = (flow_hash_config & IP_FLOW_HASH_SRC_ADDR)
    ? ip->src_address.data_u32 : 0;
  t2 = (flow_hash_config & IP_FLOW_HASH_DST_ADDR)
    ? ip->dst_address.data_u32 : 0;

  a = (flow_hash_config & IP_FLOW_HASH_REVERSE_SRC_DST) ? t2 : t1;
  b = (flow_hash_config & IP_FLOW_HASH_REVERSE_SRC_DST) ? t1 : t2;

  // tcp的话，再加上端口，udp的话由于ip分片可能存在导致无法获取udp端口。
  t1 = is_tcp_udp ? tcp->src : 0;
  t2 = is_tcp_udp ? tcp->dst : 0;

  t1 = (flow_hash_config & IP_FLOW_HASH_SRC_PORT) ? t1 : 0;
  t2 = (flow_hash_config & IP_FLOW_HASH_DST_PORT) ? t2 : 0;

  // 根据配置，如果需要流两个方向生成相同hash值的话
  if (flow_hash_config & IP_FLOW_HASH_SYMMETRIC)
  {
      if (b < a)
	{
	  c = a;
	  a = b;
	  b = c;
	}
      if (t2 < t1)
	{
	  t2 += t1;
	  t1 = t2 - t1;
	  t2 = t2 - t1;
	}
  }

  b ^= (flow_hash_config & IP_FLOW_HASH_PROTO) ? ip->protocol : 0;
  c = (flow_hash_config & IP_FLOW_HASH_REVERSE_SRC_DST) ?
    (t1 << 16) | t2 : (t2 << 16) | t1;

  hash_v3_mix32 (a, b, c);
  hash_v3_finalize32 (a, b, c);

  return c;
}
```

#### IPv6流哈希计算

IPv6流哈希计算源码位`vnet/ip/ip6.h`于文件中。

计算过程同IPv4。

```
/* Compute flow hash.  We'll use it to select which Sponge to use for this
   flow.  And other things. */
always_inline u32
ip6_compute_flow_hash (const ip6_header_t * ip,
		       flow_hash_config_t flow_hash_config)
{
  tcp_header_t *tcp;
  u64 a, b, c;
  u64 t1, t2;
  uword is_tcp_udp = 0;
  u8 protocol = ip->protocol;

  if (PREDICT_TRUE
      ((ip->protocol == IP_PROTOCOL_TCP)
       || (ip->protocol == IP_PROTOCOL_UDP)))
  {
      is_tcp_udp = 1;
      tcp = (void *) (ip + 1);
  }
  else if (ip->protocol == IP_PROTOCOL_IP6_HOP_BY_HOP_OPTIONS)
  {
      ip6_hop_by_hop_header_t *hbh = (ip6_hop_by_hop_header_t *) (ip + 1);
      if ((hbh->protocol == IP_PROTOCOL_TCP) ||
	  (hbh->protocol == IP_PROTOCOL_UDP))
	{
	  is_tcp_udp = 1;
	  tcp = (tcp_header_t *) ((u8 *) hbh + ((hbh->length + 1) << 3));
	}
      protocol = hbh->protocol;
  }

  t1 = (ip->src_address.as_u64[0] ^ ip->src_address.as_u64[1]);
  t1 = (flow_hash_config & IP_FLOW_HASH_SRC_ADDR) ? t1 : 0;

  t2 = (ip->dst_address.as_u64[0] ^ ip->dst_address.as_u64[1]);
  t2 = (flow_hash_config & IP_FLOW_HASH_DST_ADDR) ? t2 : 0;

  a = (flow_hash_config & IP_FLOW_HASH_REVERSE_SRC_DST) ? t2 : t1;
  b = (flow_hash_config & IP_FLOW_HASH_REVERSE_SRC_DST) ? t1 : t2;

  t1 = is_tcp_udp ? tcp->src : 0;
  t2 = is_tcp_udp ? tcp->dst : 0;

  t1 = (flow_hash_config & IP_FLOW_HASH_SRC_PORT) ? t1 : 0;
  t2 = (flow_hash_config & IP_FLOW_HASH_DST_PORT) ? t2 : 0;

  if (flow_hash_config & IP_FLOW_HASH_SYMMETRIC)
  {
      if (b < a)
	{
	  c = a;
	  a = b;
	  b = c;
	}
      if (t2 < t1)
	{
	  t2 += t1;
	  t1 = t2 - t1;
	  t2 = t2 - t1;
	}
  }

  b ^= (flow_hash_config & IP_FLOW_HASH_PROTO) ? protocol : 0;
  c = (flow_hash_config & IP_FLOW_HASH_REVERSE_SRC_DST) ?
    ((t1 << 16) | t2) : ((t2 << 16) | t1);

  hash_mix64 (a, b, c);
  return (u32) c;
}
```