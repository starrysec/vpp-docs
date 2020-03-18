## arp

### arp弧和特性

#### vnet/arp/arp.c

```
VNET_FEATURE_ARC_INIT (arp_feat, static) =
{
  .arc_name = "arp",
  .start_nodes = VNET_FEATURES ("arp-input"),
  .last_in_arc = "error-drop",
  .arc_index_ptr = &ethernet_arp_main.feature_arc_index,
};
```

```
VNET_FEATURE_INIT (arp_reply_feat_node, static) =
{
  .arc_name = "arp",
  .node_name = "arp-reply",
  .runs_before = VNET_FEATURES ("arp-disabled"),
};

VNET_FEATURE_INIT (arp_proxy_feat_node, static) =
{
  .arc_name = "arp",
  .node_name = "arp-proxy",
  .runs_after = VNET_FEATURES ("arp-reply"),
  .runs_before = VNET_FEATURES ("arp-disabled"),
};

VNET_FEATURE_INIT (arp_disabled_feat_node, static) =
{
  .arc_name = "arp",
  .node_name = "arp-disabled",
  .runs_before = VNET_FEATURES ("error-drop"),
};

VNET_FEATURE_INIT (arp_drop_feat_node, static) =
{
  .arc_name = "arp",
  .node_name = "error-drop",
  .runs_before = 0,	/* last feature */
};
```

### arp图节点

#### vnet/arp/arp.c

```
VLIB_REGISTER_NODE (arp_input_node, static) =
{
  .function = arp_input,
  .name = "arp-input",
  .vector_size = sizeof (u32),
  .n_errors = ETHERNET_ARP_N_ERROR,
  .error_strings = ethernet_arp_error_strings,
  .n_next_nodes = ARP_INPUT_N_NEXT,
  .next_nodes = {
    [ARP_INPUT_NEXT_DROP] = "error-drop",
    [ARP_INPUT_NEXT_DISABLED] = "arp-disabled",
  },
  .format_buffer = format_ethernet_arp_header,
  .format_trace = format_ethernet_arp_input_trace,
};

VLIB_REGISTER_NODE (arp_disabled_node, static) =
{
  .function = arp_disabled,
  .name = "arp-disabled",
  .vector_size = sizeof (u32),
  .n_errors = ARP_DISABLED_N_ERROR,
  .error_strings = arp_disabled_error_strings,
  .n_next_nodes = ARP_DISABLED_N_NEXT,
  .next_nodes = {
    [ARP_INPUT_NEXT_DROP] = "error-drop",
  },
  .format_buffer = format_ethernet_arp_header,
  .format_trace = format_ethernet_arp_input_trace,
};

VLIB_REGISTER_NODE (arp_reply_node, static) =
{
  .function = arp_reply,
  .name = "arp-reply",
  .vector_size = sizeof (u32),
  .n_errors = ETHERNET_ARP_N_ERROR,
  .error_strings = ethernet_arp_error_strings,
  .n_next_nodes = ARP_REPLY_N_NEXT,
  .next_nodes = {
    [ARP_REPLY_NEXT_DROP] = "error-drop",
    [ARP_REPLY_NEXT_REPLY_TX] = "interface-output",
  },
  .format_buffer = format_ethernet_arp_header,
  .format_trace = format_ethernet_arp_input_trace,
};
```

```
static uword
arp_input (vlib_main_t * vm, vlib_node_runtime_t * node, vlib_frame_t * frame)
{
  u32 n_left_from, next_index, *from, *to_next, n_left_to_next;
  ethernet_arp_main_t *am = &ethernet_arp_main;

  /* frame起始地址 */
  from = vlib_frame_vector_args (frame);
  /* frame中的buffer个数 */
  n_left_from = frame->n_vectors;
  /* next节点索引 */
  next_index = node->cached_next_index;

  /* 检查节点trace标志 */
  if (node->flags & VLIB_NODE_FLAG_TRACE)
    /* 收集buffer的trace信息 */
    vlib_trace_frame_buffers_only (vm, node, from, frame->n_vectors,
				   /* stride */ 1,
				   sizeof (ethernet_arp_input_trace_t));

  while (n_left_from > 0)
    {
	  /* 获取传递给下一个node的数据包将驻留的内存结构 */
      vlib_get_next_frame (vm, node, next_index, to_next, n_left_to_next);

      while (n_left_from > 0 && n_left_to_next > 0)
		{
		  /* arp头部 */
		  const ethernet_arp_header_t *arp0;
		  /* arp-input节点的next节点 */
		  arp_input_next_t next0;
		  vlib_buffer_t *p0;
		  u32 pi0, error0;

          /* 传递给next节点的buffer数据和当前frame中buffer数据向对应。 */
		  pi0 = to_next[0] = from[0];
		  from += 1;
		  to_next += 1;
		  n_left_from -= 1;
		  n_left_to_next -= 1;

          /* 通过buffer索引后去buffer */
		  p0 = vlib_get_buffer (vm, pi0);
		  /* 获取arp头 */
		  arp0 = vlib_buffer_get_current (p0);

		  /* 初始化next节点和错误信息 */
		  error0 = ETHERNET_ARP_ERROR_replies_sent;
		  next0 = ARP_INPUT_NEXT_DROP;

		  /* 检查arp头中硬件类型是否以太网类型 */
		  error0 =
			(arp0->l2_type !=
			 clib_net_to_host_u16 (ETHERNET_ARP_HARDWARE_TYPE_ethernet) ?
			 ETHERNET_ARP_ERROR_l2_type_not_ethernet : error0);
	      /* 检查arp头中协议类型是否ipv4类型 */
		  error0 =
			(arp0->l3_type !=
			 clib_net_to_host_u16 (ETHERNET_TYPE_IP4) ?
			 ETHERNET_ARP_ERROR_l3_type_not_ip4 : error0);
	      /* 检查arp中目的ip是否设置 */
		  error0 =
			(0 == arp0->ip4_over_ethernet[0].ip4.as_u32 ?
			 ETHERNET_ARP_ERROR_l3_dst_address_unset : error0);
		  
		  /* 上面三个检查都通过，这儿应该还是初始值 */
		  if (ETHERNET_ARP_ERROR_replies_sent == error0)
			{
			  /* 设置next为arp-disabled */
			  next0 = ARP_INPUT_NEXT_DISABLED;
			  /* 使能sw_if_index rx方向的弧特性，
			  在arc中指定的起始node中，必须调用vnet_feature_arc_start函数，才能正式进入feature机制业务流程，该函数会将下一跳强行指定为arc中的下一个feature（这里是arp-disabled）。 */
			  vnet_feature_arc_start (am->feature_arc_index,
						  vnet_buffer (p0)->sw_if_index[VLIB_RX],
						  &next0, p0);
			}
		  /* 检查未通过，设置p0错误 */
		  else
			p0->error = node->errors[error0];

		  /* 验证处理结果 */
		  vlib_validate_buffer_enqueue_x1 (vm, node, next_index, to_next,
						   n_left_to_next, pi0, next0);
		}
	  /* 把传递给下一个node的数据包写入特定位置。这样下一个node将正式可以被调度框架调度，并处理传给他的数据包 */
      vlib_put_next_frame (vm, node, next_index, n_left_to_next);
    }
	
  /* 返回frame中数据包个数 */
  return frame->n_vectors;
}
```

```
static uword
arp_disabled (vlib_main_t * vm,
	      vlib_node_runtime_t * node, vlib_frame_t * frame)
{
  u32 n_left_from, next_index, *from, *to_next, n_left_to_next;
  
  /* frame起始地址 */
  from = vlib_frame_vector_args (frame);
  /* frame中的buffer个数 */
  n_left_from = frame->n_vectors;
  /* next节点索引 */
  next_index = node->cached_next_index;

  /* 检查节点trace标志 */
  if (node->flags & VLIB_NODE_FLAG_TRACE)
    /* 收集buffer的trace信息 */
    vlib_trace_frame_buffers_only (vm, node, from, frame->n_vectors,
				   /* stride */ 1,
				   sizeof (ethernet_arp_input_trace_t));

  while (n_left_from > 0)
    {
	  /* 获取传递给下一个node的数据包将驻留的内存结构 */
      vlib_get_next_frame (vm, node, next_index, to_next, n_left_to_next);

      while (n_left_from > 0 && n_left_to_next > 0)
		{
		  /* arp-disabled的next节点为drop节点 */
		  arp_disabled_next_t next0 = ARP_DISABLED_NEXT_DROP;
		  vlib_buffer_t *p0;
		  u32 pi0, error0;

		  /* 初始化next节点和错误信息 */
		  next0 = ARP_DISABLED_NEXT_DROP;
		  error0 = ARP_DISABLED_ERROR_DISABLED;

		  /* 传递给next节点的buffer数据和当前frame中buffer数据向对应。next节点是drop，因此所有数据到drop节点进行drop操作并释放。 */
		  pi0 = to_next[0] = from[0];
		  from += 1;
		  to_next += 1;
		  n_left_from -= 1;
		  n_left_to_next -= 1;
		  
		  /* 通过buffer索引后去buffer */
		  p0 = vlib_get_buffer (vm, pi0);
		  p0->error = node->errors[error0];
		  
		  /* 验证处理结果 */
		  vlib_validate_buffer_enqueue_x1 (vm, node, next_index, to_next,
						   n_left_to_next, pi0, next0);
		}
		/* 把传递给下一个node的数据包写入特定位置。这样下一个node将正式可以被调度框架调度，并处理传给他的数据包 */
		vlib_put_next_frame (vm, node, next_index, n_left_to_next);
    }

  /* 返回frame中数据包个数 */
  return frame->n_vectors;
}
```

#### vnet/arp/arp_proxy.c

```

VLIB_REGISTER_NODE (arp_proxy_node, static) =
{
  .function = arp_proxy,.name = "arp-proxy",.vector_size =
    sizeof (u32),.n_errors = ETHERNET_ARP_N_ERROR,.error_strings =
    ethernet_arp_error_strings,.n_next_nodes = ARP_REPLY_N_NEXT,.next_nodes =
  {
  [ARP_REPLY_NEXT_DROP] = "error-drop",
      [ARP_REPLY_NEXT_REPLY_TX] = "interface-output",}
,.format_buffer = format_ethernet_arp_header,.format_trace =
    format_ethernet_arp_input_trace,};

static clib_error_t *
show_ip4_arp (vlib_main_t * vm,
	      unformat_input_t * input, vlib_cli_command_t * cmd)
{
  arp_proxy_main_t *am = &arp_proxy_main;
  ethernet_proxy_arp_t *pa;

  if (vec_len (am->proxy_arps))
    {
      vlib_cli_output (vm, "Proxy arps enabled for:");
      vec_foreach (pa, am->proxy_arps)
      {
	vlib_cli_output (vm, "Fib_index %d   %U - %U ",
			 pa->fib_index,
			 format_ip4_address, &pa->lo_addr,
			 format_ip4_address, &pa->hi_addr);
      }
    }

  return (NULL);
}
```

