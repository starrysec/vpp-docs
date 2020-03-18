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
			  /* 使能sw_if_index rx方向的arp弧特性，
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

```
static uword
arp_reply (vlib_main_t * vm, vlib_node_runtime_t * node, vlib_frame_t * frame)
{
  vnet_main_t *vnm = vnet_get_main ();
  u32 n_left_from, next_index, *from, *to_next;
  u32 n_replies_sent = 0;

  from = vlib_frame_vector_args (frame);
  n_left_from = frame->n_vectors;
  next_index = node->cached_next_index;

  if (node->flags & VLIB_NODE_FLAG_TRACE)
    vlib_trace_frame_buffers_only (vm, node, from, frame->n_vectors,
				   /* stride */ 1,
				   sizeof (ethernet_arp_input_trace_t));

  while (n_left_from > 0)
    {
      u32 n_left_to_next;

      vlib_get_next_frame (vm, node, next_index, to_next, n_left_to_next);

      while (n_left_from > 0 && n_left_to_next > 0)
	{
	  vlib_buffer_t *p0;
	  /* arp头 */
	  ethernet_arp_header_t *arp0;
	  /* 以太网头 */
	  ethernet_header_t *eth_rx;
	  const ip4_address_t *if_addr0;
	  u32 pi0, error0, next0, sw_if_index0, conn_sw_if_index0, fib_index0;
	  u8 dst_is_local0, is_vrrp_reply0;
	  fib_node_index_t dst_fei, src_fei;
	  const fib_prefix_t *pfx0;
	  fib_entry_flag_t src_flags, dst_flags;

	  pi0 = from[0];
	  to_next[0] = pi0;
	  from += 1;
	  to_next += 1;
	  n_left_from -= 1;
	  n_left_to_next -= 1;

	  p0 = vlib_get_buffer (vm, pi0);
	  /* 获取arp头 */
	  arp0 = vlib_buffer_get_current (p0);
	  /* 填充以太网头 */
	  eth_rx = ethernet_buffer_get_header (p0);

	  next0 = ARP_REPLY_NEXT_DROP;
	  error0 = ETHERNET_ARP_ERROR_replies_sent;
	  /* 获取po数据入接口索引 */
	  sw_if_index0 = vnet_buffer (p0)->sw_if_index[VLIB_RX];

	  /* 使用入接口索引查找路由表索引 */
	  /* Check that IP address is local and matches incoming interface. */
	  fib_index0 = ip4_fib_table_get_index_for_sw_if_index (sw_if_index0);
	  /* 未找到，错误 */
	  if (~0 == fib_index0)
	    {
	      error0 = ETHERNET_ARP_ERROR_interface_no_table;
	      goto drop;

	    }
	  /* 找到了fib索引 */
	  {
	    /*
	     * we're looking for FIB entries that indicate the source
	     * is attached. There may be more specific non-attached
	     * routes that match the source, but these do not influence
	     * whether we respond to an ARP request, i.e. they do not
	     * influence whether we are the correct way for the sender
	     * to reach us, they only affect how we reach the sender.
	     */
	    fib_entry_t *src_fib_entry;
	    const fib_prefix_t *pfx;
	    fib_entry_src_t *src;
	    fib_source_t source;
	    int attached;
	    int mask;

	    mask = 32;
	    attached = 0;

	    do
	      {
		  /* 使用arp头中的目的ip地址，在路由表中进行LPM（最长前缀）匹配 */
		src_fei = ip4_fib_table_lookup (ip4_fib_get (fib_index0),
						&arp0->
						ip4_over_ethernet[0].ip4,
						mask);
		/* 通过索引找到的路由表项 */
		src_fib_entry = fib_entry_get (src_fei);

		/*
		 * It's possible that the source that provides the
		 * flags we need, or the flags we must not have,
		 * is not the best source, so check then all.
		 */
                /* *INDENT-OFF* */
                FOR_EACH_SRC_ADDED(src_fib_entry, src, source,
                ({
				  /* 找到source源的表项标志 */
                  src_flags = fib_entry_get_flags_for_source (src_fei, source);

				  /* arp源地址是本地接口地址，拒绝回应 */
                  /* Reject requests/replies with our local interface
                     address. */
                  if (FIB_ENTRY_FLAG_LOCAL & src_flags)
                    {
                      error0 = ETHERNET_ARP_ERROR_l3_src_address_is_local;
                      /*
                       * When VPP has an interface whose address is also
                       * applied to a TAP interface on the host, then VPP's
                       * TAP interface will be unnumbered  to the 'real'
                       * interface and do proxy ARP from the host.
                       * The curious aspect of this setup is that ARP requests
                       * from the host will come from the VPP's own address.
                       * So don't drop immediately here, instead go see if this
                       * is a proxy ARP case.
                       */
                      goto next_feature;
                    }
				
                  /* arp源地址是和接口连接的对端地址，或者同一子网的地址 */				
                  /* A Source must also be local to subnet of matching
                   * interface address. */
                  if ((FIB_ENTRY_FLAG_ATTACHED & src_flags) ||
                      (FIB_ENTRY_FLAG_CONNECTED & src_flags))
                    {
                      attached = 1;
                      break;
                    }
                  /*
                   * else
                   *  The packet was sent from an address that is not
                   *  connected nor attached i.e. it is not from an
                   *  address that is covered by a link's sub-net,
                   *  nor is it a already learned host resp.
                   */
                }));
                /* *INDENT-ON* */

		/*
		 * shorter mask lookup for the next iteration.
		 */
		pfx = fib_entry_get_prefix (src_fei);
		mask = pfx->fp_len - 1;

		/*
		 * continue until we hit the default route or we find
		 * the attached we are looking for. The most likely
		 * outcome is we find the attached with the first source
		 * on the first lookup.
		 */
	      }
	    while (!attached &&
		   !fib_entry_is_sourced (src_fei, FIB_SOURCE_DEFAULT_ROUTE));

	    if (!attached)
	      {
		/*
		 * the matching route is a not attached, i.e. it was
		 * added as a result of routing, rather than interface/ARP
		 * configuration. If the matching route is not a host route
		 * (i.e. a /32)
		 */
		error0 = ETHERNET_ARP_ERROR_l3_src_address_not_local;
		goto drop;
	      }
	  }

	  dst_fei = ip4_fib_table_lookup (ip4_fib_get (fib_index0),
					  &arp0->ip4_over_ethernet[1].ip4,
					  32);
	  switch (arp_dst_fib_check (dst_fei, &dst_flags))
	    {
	    case ARP_DST_FIB_ADJ:
	      /*
	       * We matched an adj-fib on ths source subnet (a /32 previously
	       * added as a result of ARP). If this request is a gratuitous
	       * ARP, then learn from it.
	       * The check for matching an adj-fib, is to prevent hosts
	       * from spamming us with gratuitous ARPS that might otherwise
	       * blow our ARP cache
	       */
	      if (arp0->ip4_over_ethernet[0].ip4.as_u32 ==
		  arp0->ip4_over_ethernet[1].ip4.as_u32)
		error0 =
		  arp_learn (sw_if_index0, &arp0->ip4_over_ethernet[0]);
	      goto drop;
	    case ARP_DST_FIB_CONN:
	      /* destination is connected, continue to process */
	      break;
	    case ARP_DST_FIB_NONE:
	      /* destination is not connected, stop here */
	      error0 = ETHERNET_ARP_ERROR_l3_dst_address_not_local;
	      goto next_feature;
	    }

	  dst_is_local0 = (FIB_ENTRY_FLAG_LOCAL & dst_flags);
	  pfx0 = fib_entry_get_prefix (dst_fei);
	  if_addr0 = &pfx0->fp_addr.ip4;

	  is_vrrp_reply0 =
	    ((arp0->opcode ==
	      clib_host_to_net_u16 (ETHERNET_ARP_OPCODE_reply))
	     &&
	     (!memcmp
	      (arp0->ip4_over_ethernet[0].mac.bytes, vrrp_prefix,
	       sizeof (vrrp_prefix))));

	  /* Trash ARP packets whose ARP-level source addresses do not
	     match their L2-frame-level source addresses, unless it's
	     a reply from a VRRP virtual router */
	  if (!ethernet_mac_address_equal
	      (eth_rx->src_address,
	       arp0->ip4_over_ethernet[0].mac.bytes) && !is_vrrp_reply0)
	    {
	      error0 = ETHERNET_ARP_ERROR_l2_address_mismatch;
	      goto drop;
	    }

	  /* Learn or update sender's mapping only for replies to addresses
	   * that are local to the subnet */
	  if (arp0->opcode ==
	      clib_host_to_net_u16 (ETHERNET_ARP_OPCODE_reply))
	    {
	      if (dst_is_local0)
		error0 =
		  arp_learn (sw_if_index0, &arp0->ip4_over_ethernet[0]);
	      else
		/* a reply for a non-local destination could be a GARP.
		 * GARPs for hosts we know were handled above, so this one
		 * we drop */
		error0 = ETHERNET_ARP_ERROR_l3_dst_address_not_local;

	      goto next_feature;
	    }
	  else if (arp0->opcode ==
		   clib_host_to_net_u16 (ETHERNET_ARP_OPCODE_request) &&
		   (dst_is_local0 == 0))
	    {
	      goto next_feature;
	    }

	  /* Honor unnumbered interface, if any */
	  conn_sw_if_index0 = fib_entry_get_resolving_interface (dst_fei);
	  if (sw_if_index0 != conn_sw_if_index0 ||
	      sw_if_index0 != fib_entry_get_resolving_interface (src_fei))
	    {
	      /*
	       * The interface the ARP is sent to or was received on is not the
	       * interface on which the covering prefix is configured.
	       * Maybe this is a case for unnumbered.
	       */
	      if (!arp_unnumbered (p0, sw_if_index0, conn_sw_if_index0))
		{
		  error0 = ETHERNET_ARP_ERROR_unnumbered_mismatch;
		  goto drop;
		}
	    }
	  if (arp0->ip4_over_ethernet[0].ip4.as_u32 ==
	      arp0->ip4_over_ethernet[1].ip4.as_u32)
	    {
	      error0 = ETHERNET_ARP_ERROR_gratuitous_arp;
	      goto drop;
	    }

	  next0 = arp_mk_reply (vnm, p0, sw_if_index0,
				if_addr0, arp0, eth_rx);

	  /* We are going to reply to this request, so, in the absence of
	     errors, learn the sender */
	  if (!error0)
	    error0 = arp_learn (sw_if_index0, &arp0->ip4_over_ethernet[1]);

	  n_replies_sent += 1;
	  goto enqueue;

	next_feature:
	  vnet_feature_next (&next0, p0);
	  goto enqueue;

	drop:
	  p0->error = node->errors[error0];

	enqueue:
	  vlib_validate_buffer_enqueue_x1 (vm, node, next_index, to_next,
					   n_left_to_next, pi0, next0);
	}

      vlib_put_next_frame (vm, node, next_index, n_left_to_next);
    }

  vlib_error_count (vm, node->node_index,
		    ETHERNET_ARP_ERROR_replies_sent, n_replies_sent);

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

