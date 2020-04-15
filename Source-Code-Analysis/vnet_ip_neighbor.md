## ip-neighbor

ip-neighbor模块用于管理arp表。控制平面。

### 添加/删除arp缓存条目

#### vnet/ip-neighbor/ip_neighbor.c

set ip neighbor代理了老的set ip arp命令。
```
VLIB_CLI_COMMAND (ip_neighbor_command, static) = {
  .path = "set ip neighbor",
  .short_help =
  "set ip neighbor [del] <intfc> <ip-address> <mac-address> [static] [no-fib-entry] [count <count>] [fib-id <fib-id>] [proxy <lo-addr> - <hi-addr>]",
  .function = ip_neighbor_cmd,
};
VLIB_CLI_COMMAND (ip_neighbor_command2, static) = {
  .path = "ip neighbor",
  .short_help =
  "ip neighbor [del] <intfc> <ip-address> <mac-address> [static] [no-fib-entry] [count <count>] [fib-id <fib-id>] [proxy <lo-addr> - <hi-addr>]",
  .function = ip_neighbor_cmd,
};
```

arp缓存添加/删除函数输入。
```
static clib_error_t *
ip_neighbor_cmd (vlib_main_t * vm,
		 unformat_input_t * input, vlib_cli_command_t * cmd)
{
  ip46_address_t ip = ip46_address_initializer;
  mac_address_t mac = ZERO_MAC_ADDRESS;
  vnet_main_t *vnm = vnet_get_main ();
  /* neighbor标志 */
  ip_neighbor_flags_t flags;
  u32 sw_if_index = ~0;
  int is_add = 1;
  int count = 1;

  /* 初始化neighbor标志为dynamic，默认dynamic */
  flags = IP_NEIGHBOR_FLAG_DYNAMIC;

  /* 解析命令行 */
  while (unformat_check_input (input) != UNFORMAT_END_OF_INPUT)
    {
	  /* 解析接口、ip地址、mac地址 */
      /* set ip neighbor TenGigE1/1/0/1 1.2.3.4 aa:bb:... or aabb.ccdd... */
      if (unformat (input, "%U %U %U",
		    unformat_vnet_sw_interface, vnm, &sw_if_index,
		    unformat_ip46_address, &ip, IP46_TYPE_ANY,
		    unformat_mac_address_t, &mac))
	;
	  /* 是添加还是删除arp条目 */
      else if (unformat (input, "delete") || unformat (input, "del"))
		is_add = 0;
	  /* 本arp条目时static的还是dynamic的 */
      else if (unformat (input, "static"))
	{
	  /* 置static标志，清除dynamic标志 */
	  flags |= IP_NEIGHBOR_FLAG_STATIC;
	  flags &= ~IP_NEIGHBOR_FLAG_DYNAMIC;
	}
	  /* 明确指定该arp不会生成对应的路由表项 */
      else if (unformat (input, "no-fib-entry"))
		flags |= IP_NEIGHBOR_FLAG_NO_FIB_ENTRY;
	  /* count表示，从前面的ip和mac，连续添加/删除count条arp，每次ip和mac都加一 */
      else if (unformat (input, "count %d", &count))
	;
	  /* 其他 */
      else
	break;
    }

  /* 接口、IP地址、mac地址缺一不可 */
  if (sw_if_index == ~0 ||
      ip46_address_is_zero (&ip) || mac_address_is_zero (&mac))
    return clib_error_return (0,
			      "specify interface, IP address and MAC: `%U'",
			      format_unformat_error, input);
  /* 循环添加count条记录 */
  while (count)
    {
	  /* 添加arp */
      if (is_add)
	    /* 添加arp记录 */
		ip_neighbor_add (&ip, ip46_address_get_type (&ip), &mac, sw_if_index,
			 flags, NULL);
	  /* 删除arp */
      else
	    /* 删除arp记录 */
		ip_neighbor_del (&ip, ip46_address_get_type (&ip), sw_if_index);

	  /* ip地址加1 */
      ip46_address_increment (ip46_address_get_type (&ip), &ip);
	  /* mac地址加1 */
      mac_address_increment (&mac);

      --count;
    }

  return NULL;
}
```

```
int
ip_neighbor_add (const ip46_address_t * ip,
		 ip46_type_t type,
		 const mac_address_t * mac,
		 u32 sw_if_index,
		 ip_neighbor_flags_t flags, u32 * stats_index)
{
  fib_protocol_t fproto;
  ip_neighbor_t *ipn;

  /* 检查：只有主线程可以删除arp缓存 */
  /* main thread only */
  ASSERT (0 == vlib_get_thread_index ());

  /* 根据IP地址类型，确定fib协议 */
  fproto = fib_proto_from_ip46 (type);

  /* 使用ip地址、接口、ip地址类型生成key */
  const ip_neighbor_key_t key = {
    .ipnk_ip = *ip,
    .ipnk_sw_if_index = sw_if_index,
    .ipnk_type = type,
  };

  /* 使用key查询arp记录 */
  ipn = ip_neighbor_db_find (&key);

  /* 找到了，复用记录内存 */
  if (ipn)
    {
      IP_NEIGHBOR_DBG ("update: %U, %U",
		       format_vnet_sw_if_index_name, vnet_get_main (),
		       sw_if_index, format_ip46_address, ip, type,
		       format_ip_neighbor_flags, flags, format_mac_address_t,
		       mac);

      /* 已经存在的是static类型，新加的不是static类型 */
      /* Refuse to over-write static neighbor entry. */
      if (!(flags & IP_NEIGHBOR_FLAG_STATIC) &&
	  (ipn->ipn_flags & IP_NEIGHBOR_FLAG_STATIC))
	{
	  /* mac地址匹配，去进一步检查 */
	  /* if MAC address match, still check to send event */
	  if (0 == mac_address_cmp (&ipn->ipn_mac, mac))
	    goto check_customers;
	  /* mac地址不匹配，返回错误 */
	  return -2;
	}

      /*
       * prevent a DoS attack from the data-plane that
       * spams us with no-op updates to the MAC address
       */
	  /* mac地址没变，可能是来自数据平面的DoS攻击 */
      if (0 == mac_address_cmp (&ipn->ipn_mac, mac))
	{
	  /* 更新ipn时间，重新按时间排序，ipn为list第一个元素 */
	  ip_neighbor_refresh (ipn);
	  /* 去进一步检查 */
	  goto check_customers;
	}

	  /* mac地址变了，则更新mac地址 */
      mac_address_copy (&ipn->ipn_mac, mac);

      /* A dynamic entry can become static, but not vice-versa.
       * i.e. since if it was programmed by the CP then it must
       * be removed by the CP */
	  /* 已经存在的不是static类型，新加的是static类型 (dynamic可变static，反之不行)*/
      if ((flags & IP_NEIGHBOR_FLAG_STATIC) &&
	  !(ipn->ipn_flags & IP_NEIGHBOR_FLAG_STATIC))
	{
	  /* 先从list中移除ipn */
	  ip_neighbor_list_remove (ipn);
	  /* 设置arp缓存标志位static，清除原来的dynamic标志 */
	  ipn->ipn_flags |= IP_NEIGHBOR_FLAG_STATIC;
	  ipn->ipn_flags &= ~IP_NEIGHBOR_FLAG_DYNAMIC;
	}
    }
  /* 未找到，申请记录内存 */
  else
    {
      IP_NEIGHBOR_INFO ("add: %U, %U",
			format_vnet_sw_if_index_name, vnet_get_main (),
			sw_if_index, format_ip46_address, ip, type,
			format_ip_neighbor_flags, flags, format_mac_address_t,
			mac);
			
      /* 申请记录内存 */
      ipn = ip_neighbor_alloc (&key, mac, flags);
	
      if (NULL == ipn)
		return VNET_API_ERROR_LIMIT_EXCEEDED;
    }

  /* Update time stamp and flags. */
  /* 更新ipn时间，重新按时间排序，ipn为list第一个元素 */
  ip_neighbor_refresh (ipn);

  /* 根据next-hop遍历adj中的neighbo,完成后在回调ip_neighbor_mk_complete_walk中更新rewrite string */
  adj_nbr_walk_nh (ipn->ipn_key->ipnk_sw_if_index,
		   fproto, &ipn->ipn_key->ipnk_ip,
		   ip_neighbor_mk_complete_walk, ipn);

check_customers:
  /* Customer(s) requesting event for this address? */
  /* 给neighbor观察者发送事件 */
  ip_neighbor_publish (ip_neighbor_get_index (ipn));

  /* 这个参数一般为NULL，返回这个arp在adj中的索引值 */
  if (stats_index)
    *stats_index = adj_nbr_find (fproto,
				 fib_proto_to_link (fproto),
				 &ipn->ipn_key->ipnk_ip,
				 ipn->ipn_key->ipnk_sw_if_index);
  return 0;
}
```

```
int
ip_neighbor_del (const ip46_address_t * ip, ip46_type_t type, u32 sw_if_index)
{
  ip_neighbor_t *ipn;

  /* 检查：只有主线程可以删除arp缓存 */
  /* main thread only */
  ASSERT (0 == vlib_get_thread_index ());

  IP_NEIGHBOR_INFO ("delete: %U, %U",
		    format_vnet_sw_if_index_name, vnet_get_main (),
		    sw_if_index, format_ip46_address, ip, type);

  /* 使用ip地址、接口、ip地址类型生成key */
  const ip_neighbor_key_t key = {
    .ipnk_ip = *ip,
    .ipnk_sw_if_index = sw_if_index,
    .ipnk_type = type,
  };

  /* 使用key查询arp记录 */
  ipn = ip_neighbor_db_find (&key);

  /* 未找到，返回错误 */
  if (NULL == ipn)
    return (VNET_API_ERROR_NO_SUCH_ENTRY);

  /* 找到了，从neighbor db中删除记录并释放资源 */
  ip_neighbor_free (ipn);

  return (0);
}
```

### 查看arp缓存信息

```
VLIB_CLI_COMMAND (show_ip_neighbors_cmd_node, static) = {
  .path = "show ip neighbors",
  .function = ip_neighbor_show,
  .short_help = "show ip neighbors [interface]",
};
VLIB_CLI_COMMAND (show_ip4_neighbors_cmd_node, static) = {
  .path = "show ip4 neighbors",
  .function = ip4_neighbor_show,
  .short_help = "show ip4 neighbors [interface]",
};
VLIB_CLI_COMMAND (show_ip6_neighbors_cmd_node, static) = {
  .path = "show ip6 neighbors",
  .function = ip6_neighbor_show,
  .short_help = "show ip6 neighbors [interface]",
};
VLIB_CLI_COMMAND (show_ip_neighbor_cmd_node, static) = {
  .path = "show ip neighbor",
  .function = ip_neighbor_show,
  .short_help = "show ip neighbor [interface]",
};
VLIB_CLI_COMMAND (show_ip4_neighbor_cmd_node, static) = {
  .path = "show ip4 neighbor",
  .function = ip4_neighbor_show,
  .short_help = "show ip4 neighbor [interface]",
};
VLIB_CLI_COMMAND (show_ip6_neighbor_cmd_node, static) = {
  .path = "show ip6 neighbor",
  .function = ip6_neighbor_show,
  .short_help = "show ip6 neighbor [interface]",
};
VLIB_CLI_COMMAND (show_ip4_neighbor_sorted_cmd_node, static) = {
  .path = "show ip4 neighbor-sorted",
  .function = ip4_neighbor_show_sorted,
  .short_help = "show ip4 neighbor-sorted",
};
VLIB_CLI_COMMAND (show_ip6_neighbor_sorted_cmd_node, static) = {
  .path = "show ip6 neighbor-sorted",
  .function = ip6_neighbor_show_sorted,
  .short_help = "show ip6 neighbor-sorted",
};
```

### 处理（构造arp包）

adj类型：nbr（arp目的ip是数据包目的ip），glean（arp目的ip是下一跳ip），midchain（隧道情形）。

#### vnet/ip-neighbor/ip4_neighbor.c

```
VLIB_REGISTER_NODE (ip4_arp_node) =
{
  .name = "ip4-arp",
  .vector_size = sizeof (u32),
  .format_trace = format_ip4_forward_next_trace,
  .n_errors = ARRAY_LEN (ip4_arp_error_strings),
  .error_strings = ip4_arp_error_strings,
  .n_next_nodes = IP4_ARP_N_NEXT,
  .next_nodes = {
    [IP4_ARP_NEXT_DROP] = "ip4-drop",
  },
};

VLIB_REGISTER_NODE (ip4_glean_node) =
{
  .name = "ip4-glean",
  .vector_size = sizeof (u32),
  .format_trace = format_ip4_forward_next_trace,
  .n_errors = ARRAY_LEN (ip4_arp_error_strings),
  .error_strings = ip4_arp_error_strings,
  .n_next_nodes = IP4_ARP_N_NEXT,
  .next_nodes = {
    [IP4_ARP_NEXT_DROP] = "ip4-drop",
  },
};

VLIB_NODE_FN (ip4_arp_node) (vlib_main_t * vm, vlib_node_runtime_t * node,
			     vlib_frame_t * frame)
{
  return (ip4_arp_inline (vm, node, frame, 0));
}

VLIB_NODE_FN (ip4_glean_node) (vlib_main_t * vm, vlib_node_runtime_t * node,
			       vlib_frame_t * frame)
{
  return (ip4_arp_inline (vm, node, frame, 1));
}
```

```
always_inline uword
ip4_arp_inline (vlib_main_t * vm,
		vlib_node_runtime_t * node,
		vlib_frame_t * frame, int is_glean)
{
  vnet_main_t *vnm = vnet_get_main ();
  ip4_main_t *im = &ip4_main;
  ip_lookup_main_t *lm = &im->lookup_main;
  u32 *from, *to_next_drop;
  uword n_left_from, n_left_to_next_drop, next_index;
  u32 thread_index = vm->thread_index;
  u64 seed;

  if (node->flags & VLIB_NODE_FLAG_TRACE)
    ip4_forward_next_trace (vm, node, frame, VLIB_TX);

  // 初始化节流器随机种子
  seed = throttle_seed (&im->arp_throttle, thread_index, vlib_time_now (vm));

  from = vlib_frame_vector_args (frame);
  n_left_from = frame->n_vectors;
  next_index = node->cached_next_index;
  if (next_index == IP4_ARP_NEXT_DROP)
    next_index = IP4_ARP_N_NEXT;	/* point to first interface */

  while (n_left_from > 0)
  {
    vlib_get_next_frame (vm, node, IP4_ARP_NEXT_DROP,
			   to_next_drop, n_left_to_next_drop);

    while (n_left_from > 0 && n_left_to_next_drop > 0)
	{
	  u32 pi0, adj_index0, sw_if_index0;
	  ip4_address_t resolve0, src0;
	  vlib_buffer_t *p0, *b0;
	  ip_adjacency_t *adj0;
	  u64 r0;

	  pi0 = from[0];
      // 当前数据包
	  p0 = vlib_get_buffer (vm, pi0);

	  from += 1;
	  n_left_from -= 1;
	  to_next_drop[0] = pi0;
	  to_next_drop += 1;
	  n_left_to_next_drop -= 1;

      /* Adjacency from destination IP address lookup [VLIB_TX].
         Adjacency from source IP address lookup [VLIB_RX].
         This gets set to ~0 until source lookup is performed. */
      // 根据数据包目的地址，获取adj
	  adj_index0 = vnet_buffer (p0)->ip.adj_index[VLIB_TX];
	  adj0 = adj_get (adj_index0);
      // 根据adj的rewrite头，获取tx接口索引
	  sw_if_index0 = adj0->rewrite_header.sw_if_index;

      // 来自ip4-glean节点的调用
	  if (is_glean)
	  {
	      /* resolve the packet's destination */
          // 取当前数据包的目的地址作为构造的arp包的目的地址
	      ip4_header_t *ip0 = vlib_buffer_get_current (p0);
	      resolve0 = ip0->dst_address;
          // 取glean类型adj中包接收地址为构造的arp包的源地址，保证arp包发送后可以收到回包，glean的receive_addr未理解清楚（本地地址前缀或者空），待补充
	      src0 = adj0->sub_type.glean.receive_addr.ip4;
	  }
      // 来自ip4-arp节点的调用
	  else
	  {
	    /* resolve the incomplete adj */
        // 取nbr类型adj的下一跳地址为构造的arp包的目的地址
	    resolve0 = adj0->sub_type.nbr.next_hop.ip4;
	    /* Src IP address in ARP header. */
        // 取发送接口的ip地址为构造的arp包的源地址
	    if (ip4_src_address_for_packet (lm, sw_if_index0, &src0))
		{
		  /* No source address available */
		  p0->error = node->errors[IP4_ARP_ERROR_NO_SOURCE_ADDRESS];
		  continue;
		}
	  }

	  /* combine the address and interface for the hash key */
      // 使用目的ip和发送接口构造hash key
	  r0 = (u64) resolve0.data_u32 << 32;
	  r0 |= sw_if_index0;

      // 节流器检测，是否需要节流，防止发送大量arp
	  if (throttle_check (&im->arp_throttle, thread_index, r0, seed))
	  {
	    p0->error = node->errors[IP4_ARP_ERROR_THROTTLED];
	    continue;
	  }

	  /*
	   * the adj has been updated to a rewrite but the node the DPO that got
	   * us here hasn't - yet. no big deal. we'll drop while we wait.
	   */
      // ip4-lookup后的next节点，然后...待补充
	  if (IP_LOOKUP_NEXT_REWRITE == adj0->lookup_next_index)
	  {
	    p0->error = node->errors[IP4_ARP_ERROR_RESOLVED];
	    continue;
	  }

	  /*
	   * Can happen if the control-plane is programming tables
	   * with traffic flowing; at least that's today's lame excuse.
	   */
      // 校验状态：来自ip4-glean节点的调用lookup_next_index必须是IP_LOOKUP_NEXT_GLEAN，来自ip4-arp节点的调用lookup_next_index必须是IP_LOOKUP_NEXT_ARP
	  if ((is_glean && adj0->lookup_next_index != IP_LOOKUP_NEXT_GLEAN)
	      || (!is_glean && adj0->lookup_next_index != IP_LOOKUP_NEXT_ARP))
	  {
	    p0->error = node->errors[IP4_ARP_ERROR_NON_ARP_ADJ];
	    continue;
	  }

	  /* Send ARP request. */
	  b0 = ip4_neighbor_probe (vm, vnm, adj0, &src0, &resolve0);

	  if (PREDICT_TRUE (NULL != b0))
	  {
	    /* copy the persistent fields from the original */
        // 把当前数据包的不透明数据拷贝给构造的arp包的不透明字段，私有数据有私用
	    clib_memcpy_fast (b0->opaque2, p0->opaque2, sizeof (p0->opaque2));
	    p0->error = node->errors[IP4_ARP_ERROR_REQUEST_SENT];
	  }
	  else
	  {
	    p0->error = node->errors[IP4_ARP_ERROR_NO_BUFFERS];
	    continue;
	  }
	}

    // 丢弃当前frame
    vlib_put_next_frame (vm, node, IP4_ARP_NEXT_DROP, n_left_to_next_drop);
  }

  return frame->n_vectors;
}
```