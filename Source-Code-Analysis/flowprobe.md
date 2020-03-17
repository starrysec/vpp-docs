## flowprobe

### 配置

### 处理

#### plugins/flowprobe/node.c
```
/* *INDENT-OFF* */
VLIB_REGISTER_NODE (flowprobe_ip4_node) = {
  .function = flowprobe_ip4_node_fn,
  .name = "flowprobe-ip4",
  .vector_size = sizeof (u32),
  .format_trace = format_flowprobe_trace,
  .type = VLIB_NODE_TYPE_INTERNAL,
  .n_errors = ARRAY_LEN(flowprobe_error_strings),
  .error_strings = flowprobe_error_strings,
  .n_next_nodes = FLOWPROBE_N_NEXT,
  .next_nodes = FLOWPROBE_NEXT_NODES,
};
VLIB_REGISTER_NODE (flowprobe_ip6_node) = {
  .function = flowprobe_ip6_node_fn,
  .name = "flowprobe-ip6",
  .vector_size = sizeof (u32),
  .format_trace = format_flowprobe_trace,
  .type = VLIB_NODE_TYPE_INTERNAL,
  .n_errors = ARRAY_LEN(flowprobe_error_strings),
  .error_strings = flowprobe_error_strings,
  .n_next_nodes = FLOWPROBE_N_NEXT,
  .next_nodes = FLOWPROBE_NEXT_NODES,
};
VLIB_REGISTER_NODE (flowprobe_l2_node) = {
  .function = flowprobe_l2_node_fn,
  .name = "flowprobe-l2",
  .vector_size = sizeof (u32),
  .format_trace = format_flowprobe_trace,
  .type = VLIB_NODE_TYPE_INTERNAL,
  .n_errors = ARRAY_LEN(flowprobe_error_strings),
  .error_strings = flowprobe_error_strings,
  .n_next_nodes = FLOWPROBE_N_NEXT,
  .next_nodes = FLOWPROBE_NEXT_NODES,
};
VLIB_REGISTER_NODE (flowprobe_walker_node) = {
  .function = flowprobe_walker_process,
  .name = "flowprobe-walker",
  .type = VLIB_NODE_TYPE_INPUT,
  .state = VLIB_NODE_STATE_INTERRUPT,
};
/* *INDENT-ON* */
```

```
uword
flowprobe_node_fn (vlib_main_t * vm,
		   vlib_node_runtime_t * node, vlib_frame_t * frame,
		   flowprobe_variant_t which)
{
  u32 n_left_from, *from, *to_next;
  flowprobe_next_t next_index;
  flowprobe_main_t *fm = &flowprobe_main;
  timestamp_nsec_t timestamp;

  unix_time_now_nsec_fraction (&timestamp.sec, &timestamp.nsec);

  /* frame起始地址 */
  from = vlib_frame_vector_args (frame);
  /* frame中数据包个数 */
  n_left_from = frame->n_vectors;
  /* flowprobe节点之后的处理节点 */
  next_index = node->cached_next_index;

  while (n_left_from > 0)
    {
      u32 n_left_to_next;

      /* vlib_get_next_frame，vlib_put_next_frame几乎每个node中必定出现的一对好基友。 vlib_get_next_frame获取传递给下一个node的数据包将驻留的内存结构。vlib_put_next_frame把传递给下一个node的数据包写入特定位置。
	  这样下一个node将正式可以被调度框架调度，并处理传给他的数据包
	  */
      vlib_get_next_frame (vm, node, next_index, to_next, n_left_to_next);

      while (n_left_from >= 4 && n_left_to_next >= 2)
	{
	  /* 初始化next节点为drop节点 */
	  u32 next0 = FLOWPROBE_NEXT_DROP;
	  u32 next1 = FLOWPROBE_NEXT_DROP;
	  u16 len0, len1;
	  u32 bi0, bi1;
	  vlib_buffer_t *b0, *b1;

	  /* 从缓存行中预取后面两个数据包 */
	  /* Prefetch next iteration. */
	  {
	    vlib_buffer_t *p2, *p3;

	    p2 = vlib_get_buffer (vm, from[2]);
	    p3 = vlib_get_buffer (vm, from[3]);

	    vlib_prefetch_buffer_header (p2, LOAD);
	    vlib_prefetch_buffer_header (p3, LOAD);

	    CLIB_PREFETCH (p2->data, CLIB_CACHE_LINE_BYTES, STORE);
	    CLIB_PREFETCH (p3->data, CLIB_CACHE_LINE_BYTES, STORE);
	  }

      /* 数据包索引放入传递给下个node的结构中 */
	  /* speculatively enqueue b0 and b1 to the current next frame */
	  to_next[0] = bi0 = from[0];
	  to_next[1] = bi1 = from[1];
	  from += 2;
	  to_next += 2;
	  n_left_from -= 2;
	  n_left_to_next -= 2;

	  /* 根据数据包索引，转换成实际的指针 */
	  b0 = vlib_get_buffer (vm, bi0);
	  b1 = vlib_get_buffer (vm, bi1);

	  /* 根据配置获取next节点 */
	  vnet_feature_next (&next0, b0);
	  vnet_feature_next (&next1, b1);

	  /* 获取b0所在缓存链中总数据长度 */
	  len0 = vlib_buffer_length_in_chain (vm, b0);
	  /* 获取b0的以太网头 */
	  ethernet_header_t *eh0 = vlib_buffer_get_current (b0);
	  /* 获取b0的以太网类型 */
	  u16 ethertype0 = clib_net_to_host_u16 (eh0->type);

	  /* 数据包是需要提取ipfix的数据包 */
	  if (PREDICT_TRUE ((b0->flags & VNET_BUFFER_F_FLOW_REPORT) == 0))
	    /* 提取ipfix信息 */
	    add_to_flow_record_state (vm, node, fm, b0, timestamp, len0,
				      flowprobe_get_variant
				      (which, fm->context[which].flags,
				       ethertype0), 0);

      /* 获取b1所在缓存链中总数据长度 */
	  len1 = vlib_buffer_length_in_chain (vm, b1);
	  /* 获取b1的以太网头 */
	  ethernet_header_t *eh1 = vlib_buffer_get_current (b1);
	  /* 获取b0的以太网类型 */
	  u16 ethertype1 = clib_net_to_host_u16 (eh1->type);

	  /* 数据包是需要提取ipfix的数据包 */
	  if (PREDICT_TRUE ((b1->flags & VNET_BUFFER_F_FLOW_REPORT) == 0))
		/* 提取ipfix信息 */
	    add_to_flow_record_state (vm, node, fm, b1, timestamp, len1,
				      flowprobe_get_variant
				      (which, fm->context[which].flags,
				       ethertype1), 0);

	  /* 验证处理结果，当next0 == next_index时，说明该包被正确处理，该宏将do nothing
	     否则，说明本来该包应去next_index但是发生错误，使得next0 != next_index。该宏会将该错误包索引bi0发往到next0实际的下一个结点(next_index) */
	  /* verify speculative enqueues, maybe switch current next frame */
	  vlib_validate_buffer_enqueue_x2 (vm, node, next_index,
					   to_next, n_left_to_next,
					   bi0, bi1, next0, next1);
	}

      while (n_left_from > 0 && n_left_to_next > 0)
	{
	  u32 bi0;
	  vlib_buffer_t *b0;
	  u32 next0 = FLOWPROBE_NEXT_DROP;
	  u16 len0;

	  /* speculatively enqueue b0 to the current next frame */
	  bi0 = from[0];
	  to_next[0] = bi0;
	  from += 1;
	  to_next += 1;
	  n_left_from -= 1;
	  n_left_to_next -= 1;

	  b0 = vlib_get_buffer (vm, bi0);

	  vnet_feature_next (&next0, b0);

	  len0 = vlib_buffer_length_in_chain (vm, b0);
	  ethernet_header_t *eh0 = vlib_buffer_get_current (b0);
	  u16 ethertype0 = clib_net_to_host_u16 (eh0->type);

	  if (PREDICT_TRUE ((b0->flags & VNET_BUFFER_F_FLOW_REPORT) == 0))
	    {
	      flowprobe_trace_t *t = 0;
	      if (PREDICT_FALSE ((node->flags & VLIB_NODE_FLAG_TRACE)
				 && (b0->flags & VLIB_BUFFER_IS_TRACED)))
		t = vlib_add_trace (vm, node, b0, sizeof (*t));

	      add_to_flow_record_state (vm, node, fm, b0, timestamp, len0,
					flowprobe_get_variant
					(which, fm->context[which].flags,
					 ethertype0), t);
	    }

	  /* verify speculative enqueue, maybe switch current next frame */
	  vlib_validate_buffer_enqueue_x1 (vm, node, next_index,
					   to_next, n_left_to_next,
					   bi0, next0);
	}
	  /* 所有流程都正确处理完毕后，下一结点的frame上已经有本结点处理过后的数据索引
         执行该函数，将相关信息登记到vlib_pending_frame_t中，准备开始调度处理 
	  */
      vlib_put_next_frame (vm, node, next_index, n_left_to_next);
    }
  /* 返回frame中数据包个数 */
  return frame->n_vectors;
}
```

```
/* Per worker process processing the active/passive expired entries */
static uword
flowprobe_walker_process (vlib_main_t * vm,
			  vlib_node_runtime_t * rt, vlib_frame_t * f)
{
  flowprobe_main_t *fm = &flowprobe_main;
  flow_report_main_t *frm = &flow_report_main;
  flowprobe_entry_t *e;

  /*
   * $$$$ Remove this check from here and track FRM status and disable
   * this process if required.
   */
  if (frm->ipfix_collector.as_u32 == 0 || frm->src_address.as_u32 == 0)
    {
      fm->disabled = true;
      return 0;
    }
  fm->disabled = false;

  u32 cpu_index = os_get_thread_index ();
  u32 *to_be_removed = 0, *i;

  /*
   * Tick the timer when required and process the vector of expired
   * timers
   */
  f64 start_time = vlib_time_now (vm);
  u32 count = 0;

  tw_timer_expire_timers_2t_1w_2048sl (fm->timers_per_worker[cpu_index],
				       start_time);

  vec_foreach (i, fm->expired_passive_per_worker[cpu_index])
  {
    u32 exported = 0;
    f64 now = vlib_time_now (vm);
    if (now > start_time + 100e-6
	|| exported > FLOW_MAXIMUM_EXPORT_ENTRIES - 1)
      break;

    if (pool_is_free_index (fm->pool_per_worker[cpu_index], *i))
      {
	clib_warning ("Element is %d is freed already\n", *i);
	continue;
      }
    else
      e = pool_elt_at_index (fm->pool_per_worker[cpu_index], *i);

    /* Check last update timestamp. If it is longer than passive time nuke
     * entry. Otherwise restart timer with what's left
     * Premature passive timer by more than 10%
     */
    if ((now - e->last_updated) < (u64) (fm->passive_timer * 0.9))
      {
	u64 delta = fm->passive_timer - (now - e->last_updated);
	e->passive_timer_handle = tw_timer_start_2t_1w_2048sl
	  (fm->timers_per_worker[cpu_index], *i, 0, delta);
      }
    else			/* Nuke entry */
      {
	vec_add1 (to_be_removed, *i);
      }
    /* If anything to report send it to the exporter */
    if (e->packetcount && now > e->last_exported + fm->active_timer)
      {
	exported++;
	flowprobe_export_entry (vm, e);
      }
    count++;
  }
  if (count)
    vec_delete (fm->expired_passive_per_worker[cpu_index], count, 0);

  vec_foreach (i, to_be_removed) flowprobe_delete_by_index (cpu_index, *i);
  vec_free (to_be_removed);

  return 0;
}
```

```
static inline void
add_to_flow_record_state (vlib_main_t * vm, vlib_node_runtime_t * node,
			  flowprobe_main_t * fm, vlib_buffer_t * b,
			  timestamp_nsec_t timestamp, u16 length,
			  flowprobe_variant_t which, flowprobe_trace_t * t)
{
  /* flowprobe插件是否使能 */
  if (fm->disabled)
    return;

  /* 获取cpu索引，等于线程索引 */
  u32 my_cpu_number = vm->thread_index;
  /* 字节数 */
  u16 octets = 0;

  /* 标志，l2, l3, l4, l2_ip4, l2_ip6 */
  flowprobe_record_t flags = fm->context[which].flags;
  /* 是否收集ip4和ip6信息 */
  bool collect_ip4 = false, collect_ip6 = false;
  ASSERT (b);
  /* 数据包以太网头 */
  ethernet_header_t *eth = vlib_buffer_get_current (b);
  /* 数据包以太网类型 */
  u16 ethertype = clib_net_to_host_u16 (eth->type);
  /*  flowprobe key，包含
	  u32 rx_sw_if_index;
	  u32 tx_sw_if_index;
	  u8 src_mac[6];
	  u8 dst_mac[6];
	  u16 ethertype;
	  ip46_address_t src_address;
	  ip46_address_t dst_address;
	  u8 protocol;
	  u16 src_port;
	  u16 dst_port;
	  flowprobe_variant_t which;
  字段 */
  flowprobe_key_t k = {};

  ip4_header_t *ip4 = 0;
  ip6_header_t *ip6 = 0;
  udp_header_t *udp = 0;
  tcp_header_t *tcp = 0;
  u8 tcp_flags = 0;

  /* 用户配置了收集L3或者L4信息 */
  if (flags & FLOW_RECORD_L3 || flags & FLOW_RECORD_L4)
    {
	  /* 且数据包含有三层ipv4头，则收集ip4信息 */
      collect_ip4 = which == FLOW_VARIANT_L2_IP4 || which == FLOW_VARIANT_IP4;
	  /* 且数据包含有三层ipv4头，则收集ip6信息 */
      collect_ip6 = which == FLOW_VARIANT_L2_IP6 || which == FLOW_VARIANT_IP6;
    }
  
  /* 填充key中的接收和发送接口 */
  k.rx_sw_if_index = vnet_buffer (b)->sw_if_index[VLIB_RX];
  k.tx_sw_if_index = vnet_buffer (b)->sw_if_index[VLIB_TX];

  /* 填充key中which标志，是否包含三层头部数据 */
  k.which = which;

  /* 用户配置了收集L2信息 */
  if (flags & FLOW_RECORD_L2)
    {
      clib_memcpy_fast (k.src_mac, eth->src_address, 6);
      clib_memcpy_fast (k.dst_mac, eth->dst_address, 6);
      k.ethertype = ethertype;
    }
  /* 需要收集ipv6信息 */
  if (collect_ip6 && ethertype == ETHERNET_TYPE_IP6)
    {
      ip6 = (ip6_header_t *) (eth + 1);
	  /* 用户配置了收集L3信息 */
      if (flags & FLOW_RECORD_L3)
	{
	  /* 收集源ip和目的ip */
	  k.src_address.as_u64[0] = ip6->src_address.as_u64[0];
	  k.src_address.as_u64[1] = ip6->src_address.as_u64[1];
	  k.dst_address.as_u64[0] = ip6->dst_address.as_u64[0];
	  k.dst_address.as_u64[1] = ip6->dst_address.as_u64[1];
	}
	  /* 收集上层协议 */
      k.protocol = ip6->protocol;
	  /* 获取udp头 */
      if (k.protocol == IP_PROTOCOL_UDP)
	udp = (udp_header_t *) (ip6 + 1);
	  /* 获取tcp头 */
      else if (k.protocol == IP_PROTOCOL_TCP)
	tcp = (tcp_header_t *) (ip6 + 1);
      /* 收集载荷长度 */
      octets = clib_net_to_host_u16 (ip6->payload_length)
	+ sizeof (ip6_header_t);
    }
  /* 需要收集ipv4信息 */
  if (collect_ip4 && ethertype == ETHERNET_TYPE_IP4)
    {
      ip4 = (ip4_header_t *) (eth + 1);
	  /* 用户配置了收集L3信息 */
      if (flags & FLOW_RECORD_L3)
	{
	 /* 收集源ip和目的ip */
	  k.src_address.ip4.as_u32 = ip4->src_address.as_u32;
	  k.dst_address.ip4.as_u32 = ip4->dst_address.as_u32;
	}
	/* 收集上层协议 */
      k.protocol = ip4->protocol;
	  /* 获取udp头 */
      if ((flags & FLOW_RECORD_L4) && k.protocol == IP_PROTOCOL_UDP)
	udp = (udp_header_t *) (ip4 + 1);
	/* 获取tcp头 */
      else if ((flags & FLOW_RECORD_L4) && k.protocol == IP_PROTOCOL_TCP)
	tcp = (tcp_header_t *) (ip4 + 1);
	  /* 收集载荷长度 */
      octets = clib_net_to_host_u16 (ip4->length);
    }

  /* 有udp头，则收集udp信息 */
  if (udp)
    {
      k.src_port = udp->src_port;
      k.dst_port = udp->dst_port;
    }
  /* 有tcp头，则收集tcp信息 */
  else if (tcp)
    {
      k.src_port = tcp->src_port;
      k.dst_port = tcp->dst_port;
      tcp_flags = tcp->flags;
    }

  /* 开启trace的话，填充flowprobe的trace信息 */
  if (t)
    {
      t->rx_sw_if_index = k.rx_sw_if_index;
      t->tx_sw_if_index = k.tx_sw_if_index;
      clib_memcpy_fast (t->src_mac, k.src_mac, 6);
      clib_memcpy_fast (t->dst_mac, k.dst_mac, 6);
      t->ethertype = k.ethertype;
      t->src_address.ip4.as_u32 = k.src_address.ip4.as_u32;
      t->dst_address.ip4.as_u32 = k.dst_address.ip4.as_u32;
      t->protocol = k.protocol;
      t->src_port = k.src_port;
      t->dst_port = k.dst_port;
      t->which = k.which;
    }

  /* 初始化flowprobe条目 */
  flowprobe_entry_t *e = 0;
  /* 获取当前时间 */
  f64 now = vlib_time_now (vm);
  /* 配置了active_timer，且大于0 */
  if (fm->active_timer > 0)
    {
      u32 poolindex = ~0;
      bool collision = false;
		
	  /* 查找flowprobe条目 */
      e = flowprobe_lookup (my_cpu_number, &k, &poolindex, &collision);
	  /* 查找结果为冲突时 */
      if (collision)
		{
		  /* 有数据则立即导出 */
		  /* Flush data and clean up entry for reuse. */
		  if (e->packetcount)
			flowprobe_export_entry (vm, e);
		  /* 重新利用,初始化key和flow起始时间 */
		  e->key = k;
		  e->flow_start = timestamp;
		  /* 更新冲突计数器 */
		  vlib_node_increment_counter (vm, node->node_index,
						   FLOWPROBE_ERROR_COLLISION, 1);
		}
	  /* 如果flowprobe条目不存在，则新建 */
      if (!e)			/* Create new entry */
		{
		  /* 创建flowprobe条目 */
		  e = flowprobe_create (my_cpu_number, &k, &poolindex);
		  /* 初始化上次导出时间 */
		  e->last_exported = now;
		  /* 设置flow起始时间 */
		  e->flow_start = timestamp;
		}
    }
  /* active_timer为0(即未配置)，
  else
    {
	  /* 则从stateless_entry分配flowprobe条目 */
      e = &fm->stateless_entry[my_cpu_number];
	  /* 初始化key */
      e->key = k;
    }

  /* flowprobe条目存在，则更新动态数据 */
  if (e)
    {
      /* Updating entry */
	  /* 更新数据包个数 */
      e->packetcount++;
	  /* 更新字节数 */
      e->octetcount += octets;
	  /* 更新最近更新时间 */
      e->last_updated = now;
	  /* 更新flow终止时间 */
      e->flow_end = timestamp;
	  /* 更新tcp flags */
      e->prot.tcp.flags |= tcp_flags;
	  /* 如果active_timer为0(即未配置)，或者到了主动导出时间，则导出flowprobe条目 */
      if (fm->active_timer == 0 || (now > e->last_exported + fm->active_timer))
	    /* 导出flowprobe条目 */
		flowprobe_export_entry (vm, e);
    }
}
```

```
static void
flowprobe_export_entry (vlib_main_t * vm, flowprobe_entry_t * e)
{
  u32 my_cpu_number = vm->thread_index;
  flowprobe_main_t *fm = &flowprobe_main;
  flow_report_main_t *frm = &flow_report_main;
  /* 数据缓存区 */
  vlib_buffer_t *b0;
  bool collect_ip4 = false, collect_ip6 = false;
  flowprobe_variant_t which = e->key.which;
  flowprobe_record_t flags = fm->context[which].flags;
  /* 数据包偏移量 */
  u16 offset =
    fm->context[which].next_record_offset_per_worker[my_cpu_number];

  /* 获取ipfix头长度（ip头+udp头+ipfix头），并修正offset */
  if (offset < flowprobe_get_headersize ())
    offset = flowprobe_get_headersize ();

  /* 分配数据缓冲区 */
  b0 = flowprobe_get_buffer (vm, which);
  /* No available buffer, what to do... */
  if (b0 == 0)
    return;

  /* 用户配置了收集L3信息 */
  if (flags & FLOW_RECORD_L3)
    {
	  /* 且数据包含有三层ipv4头，则收集ip4信息 */
      collect_ip4 = which == FLOW_VARIANT_L2_IP4 || which == FLOW_VARIANT_IP4;
	  /* 且数据包含有三层ipv6头，则收集ip6信息 */
      collect_ip6 = which == FLOW_VARIANT_L2_IP6 || which == FLOW_VARIANT_IP6;
    }
  /* 修正offset */
  offset += flowprobe_common_add (b0, e, offset);

  /* 用户配置了收集L2信息 */
  if (flags & FLOW_RECORD_L2)
    /* 构造L2数据，修正offset */
    offset += flowprobe_l2_add (b0, e, offset);
  /* 需要收集ipv4信息 */
  if (collect_ip6)
    /* 构造ipv4数据，修正offset */
    offset += flowprobe_l3_ip6_add (b0, e, offset);
  /* 需要收集ipv4信息 */
  if (collect_ip4)
    /* 构造ipv6数据，修正offset */
    offset += flowprobe_l3_ip4_add (b0, e, offset);
  /* 用户配置了收集L4信息 */
  if (flags & FLOW_RECORD_L4)
    /* 构造L4数据，修正offset */
    offset += flowprobe_l4_add (b0, e, offset);

  /* 重置flowprobe条目 */
  /* Reset per flow-export counters */
  e->packetcount = 0;
  e->octetcount = 0;
  e->last_exported = vlib_time_now (vm);

  /* 更新缓冲区长度 */
  b0->current_length = offset;
  /* 记录offset */
  fm->context[which].next_record_offset_per_worker[my_cpu_number] = offset;
  /* 缓冲区长度+ipfix头部模板长度超过了mtu限制，则发送此数据 */
  /* Time to flush the buffer? */
  if (offset + fm->template_size[flags] > frm->path_mtu)
    flowprobe_export_send (vm, b0, which);
}
```

```
static void
flowprobe_export_send (vlib_main_t * vm, vlib_buffer_t * b0,
		       flowprobe_variant_t which)
{
  flowprobe_main_t *fm = &flowprobe_main;
  flow_report_main_t *frm = &flow_report_main;
  vlib_frame_t *f;
  /* 包含ip4_header_t, udp_header_t, ipfix_template_packet_t */
  ip4_ipfix_template_packet_t *tp;
  ipfix_set_header_t *s;
  ipfix_message_header_t *h;
  ip4_header_t *ip;
  udp_header_t *udp;
  flowprobe_record_t flags = fm->context[which].flags;
  u32 my_cpu_number = vm->thread_index;

  /* Fill in header */
  flow_report_stream_t *stream;

  /* Nothing to send */
  if (fm->context[which].next_record_offset_per_worker[my_cpu_number] <=
      flowprobe_get_headersize ())
    return;

  u32 i, index = vec_len (frm->streams);
  for (i = 0; i < index; i++)
    if (frm->streams[i].domain_id == 1)
      {
	index = i;
	break;
      }
  if (i == vec_len (frm->streams))
    {
      vec_validate (frm->streams, index);
      frm->streams[index].domain_id = 1;
    }
  stream = &frm->streams[index];

  tp = vlib_buffer_get_current (b0);
  ip = (ip4_header_t *) & tp->ip4;
  udp = (udp_header_t *) (ip + 1);
  h = (ipfix_message_header_t *) (udp + 1);
  s = (ipfix_set_header_t *) (h + 1);

  /* 填充ifpix数据包的ip头和udp头 */
  ip->ip_version_and_header_length = 0x45;
  ip->ttl = 254;
  ip->protocol = IP_PROTOCOL_UDP;
  ip->flags_and_fragment_offset = 0;
  ip->src_address.as_u32 = frm->src_address.as_u32;
  ip->dst_address.as_u32 = frm->ipfix_collector.as_u32;
  udp->src_port = clib_host_to_net_u16 (UDP_DST_PORT_ipfix);
  udp->dst_port = clib_host_to_net_u16 (UDP_DST_PORT_ipfix);
  udp->checksum = 0;

  /* FIXUP: message header export_time */
  h->export_time = (u32)
    (((f64) frm->unix_time_0) +
     (vlib_time_now (frm->vlib_main) - frm->vlib_time_0));
  h->export_time = clib_host_to_net_u32 (h->export_time);
  h->domain_id = clib_host_to_net_u32 (stream->domain_id);

  /* FIXUP: message header sequence_number */
  h->sequence_number = stream->sequence_number++;
  h->sequence_number = clib_host_to_net_u32 (h->sequence_number);

  s->set_id_length = ipfix_set_id_length (fm->template_reports[flags],
					  b0->current_length -
					  (sizeof (*ip) + sizeof (*udp) +
					   sizeof (*h)));
  h->version_length = version_length (b0->current_length -
				      (sizeof (*ip) + sizeof (*udp)));

  ip->length = clib_host_to_net_u16 (b0->current_length);

  ip->checksum = ip4_header_checksum (ip);
  udp->length = clib_host_to_net_u16 (b0->current_length - sizeof (*ip));

  /* 计算udp校验和 */
  if (frm->udp_checksum)
    {
      /* RFC 7011 section 10.3.2. */
      udp->checksum = ip4_tcp_udp_compute_checksum (vm, b0, ip);
      if (udp->checksum == 0)
	udp->checksum = 0xffff;
    }

  ASSERT (ip->checksum == ip4_header_checksum (ip));

  /* Find or allocate a frame */
  f = fm->context[which].frames_per_worker[my_cpu_number];
  if (PREDICT_FALSE (f == 0))
    {
      u32 *to_next;
      f = vlib_get_frame_to_node (vm, ip4_lookup_node.index);
      fm->context[which].frames_per_worker[my_cpu_number] = f;
      u32 bi0 = vlib_get_buffer_index (vm, b0);

      /* Enqueue the buffer */
      to_next = vlib_frame_vector_args (f);
      to_next[0] = bi0;
      f->n_vectors = 1;
    }

  /* frame发往下一node，进行路由后发出 */
  vlib_put_frame_to_node (vm, ip4_lookup_node.index, f);
  /* 更新导出数据包计数器 */
  vlib_node_increment_counter (vm, flowprobe_l2_node.index,
			       FLOWPROBE_ERROR_EXPORTED_PACKETS, 1);

  /* 重置 */
  fm->context[which].frames_per_worker[my_cpu_number] = 0;
  fm->context[which].buffers_per_worker[my_cpu_number] = 0;
  fm->context[which].next_record_offset_per_worker[my_cpu_number] =
    flowprobe_get_headersize ();
}
```