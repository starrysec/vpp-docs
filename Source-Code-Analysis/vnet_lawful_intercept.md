## 合法监听

合法监听功能是根据美国法律，也被照搬到许多其它国家，尤其是欧洲，所有的电信设备都必须提供设施用于对被选中的用户的呼叫进行监听。必须有一定程度的支撑，以便它能被建立在任何不同的网元中。按照相关的美国法律，“合法监听”的概念也被称为“通信协助执法”（CommunicationsAssistanceforLawEnforcementAct，简称：CALEA）。
基本上，合法监听的实现和电话会议的实现类似。当A和B在相互谈话时，C可以进入这个通话，并静默地收听。

### 配置

源代码位于`vnet/lawful-intercept/lawful-intercept.c`文件中。

```
VLIB_CLI_COMMAND (set_li_command, static) = {
    .path = "set li",
    .short_help =
    "set li src <ip4-address> collector <ip4-address> udp-port <nnnn>",
    .function = set_li_command_fn,
};
```

```
static clib_error_t *
set_li_command_fn (vlib_main_t * vm,
		   unformat_input_t * input, vlib_cli_command_t * cmd)
{
  li_main_t *lm = &li_main;
  ip4_address_t collector;
  u8 collector_set = 0;
  ip4_address_t src;
  u8 src_set = 0;
  u32 tmp;
  u16 udp_port = 0;
  u8 is_add = 1;
  int i;

  while (unformat_check_input (input) != UNFORMAT_END_OF_INPUT)
  {
    // 解析collector ip地址
    if (unformat (input, "collector %U", unformat_ip4_address, &collector))
      collector_set = 1;
    // 设置源ip地址
    if (unformat (input, "src %U", unformat_ip4_address, &src))
      src_set = 1;
    // 解析collector udp端口
    else if (unformat (input, "udp-port %d", &tmp))
      udp_port = tmp;
    // 新建or删除
    else if (unformat (input, "del"))
      is_add = 0;
    else
      break;
  }

  // collector ip，port和src ip必须设置
  if (collector_set == 0)
    return clib_error_return (0, "collector must be set...");
  if (src_set == 0)
    return clib_error_return (0, "src must be set...");
  if (udp_port == 0)
    return clib_error_return (0, "udp-port must be set...");

  // 新增
  if (is_add == 1)
  {
    for (i = 0; i < vec_len (lm->collectors); i++)
	{
      // collector ip地址或者端口已经配置了
	  if (lm->collectors[i].as_u32 == collector.as_u32)
	  {
	    if (lm->ports[i] == udp_port)
	      return clib_error_return (0, "collector %U:%d already configured", &collector, udp_port);
	    else
	      return clib_error_return (0, "collector %U already configured with port %d", &collector, (int) (lm->ports[i]));
	  }
	}
    // 添加collector ip和端口，src ip到vector
    vec_add1 (lm->collectors, collector);
    vec_add1 (lm->ports, udp_port);
    vec_add1 (lm->src_addrs, src);
    return 0;
  }
  // 删除
  else
  {
    // 从vector中删除collector
    for (i = 0; i < vec_len (lm->collectors); i++)
	{
	  if ((lm->collectors[i].as_u32 == collector.as_u32)
	      && lm->ports[i] == udp_port)
	    {
	      vec_delete (lm->collectors, 1, i);
	      vec_delete (lm->ports, 1, i);
	      vec_delete (lm->src_addrs, 1, i);
	      return 0;
	    }
	}
    return clib_error_return (0, "collector %U:%d not configured", &collector, udp_port);
  }
  return 0;
}
```

### 处理

源代码位于`vnet/lawful-intercept/node.c`文件中。

```
VLIB_REGISTER_NODE (li_hit_node) = {
  .name = "li-hit",
  .vector_size = sizeof (u32),
  .format_trace = format_li_hit_trace,
  .type = VLIB_NODE_TYPE_INTERNAL,

  .n_errors = ARRAY_LEN(li_hit_error_strings),
  .error_strings = li_hit_error_strings,

  .n_next_nodes = LI_HIT_N_NEXT,

  /* edit / add dispositions here */
  .next_nodes = {
        [LI_HIT_NEXT_ETHERNET] = "ethernet-input-not-l2",
  },
};
```

```

VLIB_NODE_FN (li_hit_node) (vlib_main_t * vm,
			    vlib_node_runtime_t * node, vlib_frame_t * frame)
{
  u32 n_left_from, *from, *to_next;
  li_hit_next_t next_index;
  vlib_frame_t *int_frame = 0;
  u32 *to_int_next = 0;
  li_main_t *lm = &li_main;

  from = vlib_frame_vector_args (frame);
  n_left_from = frame->n_vectors;
  next_index = node->cached_next_index;

  if (PREDICT_FALSE (vec_len (lm->collectors) == 0))
  {
    vlib_node_increment_counter (vm, li_hit_node.index,
				   LI_HIT_ERROR_NO_COLLECTOR, n_left_from);
  }
  else
  {
    /* The intercept frame... */
    int_frame = vlib_get_frame_to_node (vm, ip4_lookup_node.index);
    to_int_next = vlib_frame_vector_args (int_frame);
  }

  while (n_left_from > 0)
  {
      u32 n_left_to_next;

      vlib_get_next_frame (vm, node, next_index, to_next, n_left_to_next);

#if 0
    while (n_left_from >= 4 && n_left_to_next >= 2)
	{
	  u32 next0 = LI_HIT_NEXT_INTERFACE_OUTPUT;
	  u32 next1 = LI_HIT_NEXT_INTERFACE_OUTPUT;
	  u32 sw_if_index0, sw_if_index1;
	  u8 tmp0[6], tmp1[6];
	  ethernet_header_t *en0, *en1;
	  u32 bi0, bi1;
	  vlib_buffer_t *b0, *b1;

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

	  /* speculatively enqueue b0 and b1 to the current next frame */
	  to_next[0] = bi0 = from[0];
	  to_next[1] = bi1 = from[1];
	  from += 2;
	  to_next += 2;
	  n_left_from -= 2;
	  n_left_to_next -= 2;

	  b0 = vlib_get_buffer (vm, bi0);
	  b1 = vlib_get_buffer (vm, bi1);

	  /* $$$$$ Dual loop: process 2 x packets here $$$$$ */
	  ASSERT (b0->current_data == 0);
	  ASSERT (b1->current_data == 0);

	  en0 = vlib_buffer_get_current (b0);
	  en1 = vlib_buffer_get_current (b1);

	  sw_if_index0 = vnet_buffer (b0)->sw_if_index[VLIB_RX];
	  sw_if_index1 = vnet_buffer (b1)->sw_if_index[VLIB_RX];

	  /* Send pkt back out the RX interface */
	  vnet_buffer (b0)->sw_if_index[VLIB_TX] = sw_if_index0;
	  vnet_buffer (b1)->sw_if_index[VLIB_TX] = sw_if_index1;

	  /* $$$$$ End of processing 2 x packets $$$$$ */

	  if (PREDICT_FALSE ((node->flags & VLIB_NODE_FLAG_TRACE)))
	    {
	      if (b0->flags & VLIB_BUFFER_IS_TRACED)
		{
		  li_hit_trace_t *t =
		    vlib_add_trace (vm, node, b0, sizeof (*t));
		  t->sw_if_index = sw_if_index0;
		  t->next_index = next0;
		}
	      if (b1->flags & VLIB_BUFFER_IS_TRACED)
		{
		  li_hit_trace_t *t =
		    vlib_add_trace (vm, node, b1, sizeof (*t));
		  t->sw_if_index = sw_if_index1;
		  t->next_index = next1;
		}
	    }

	  /* verify speculative enqueues, maybe switch current next frame */
	  vlib_validate_buffer_enqueue_x2 (vm, node, next_index,
					   to_next, n_left_to_next,
					   bi0, bi1, next0, next1);
	}
#endif /* $$$ dual-loop off */

    while (n_left_from > 0 && n_left_to_next > 0)
	{
	  u32 bi0;
	  vlib_buffer_t *b0;
	  vlib_buffer_t *c0;
	  ip4_udp_header_t *iu0;
	  ip4_header_t *ip0;
	  udp_header_t *udp0;
	  u32 next0 = LI_HIT_NEXT_ETHERNET;

	  /* speculatively enqueue b0 to the current next frame */
	  bi0 = from[0];
	  to_next[0] = bi0;
	  from += 1;
	  to_next += 1;
	  n_left_from -= 1;
	  n_left_to_next -= 1;

	  b0 = vlib_get_buffer (vm, bi0);
	  if (PREDICT_TRUE (to_int_next != 0))
	  {
	    /* Make an intercept copy. This can fail. */
        // 为了，拷贝一份数据包
	    c0 = vlib_buffer_copy (vm, b0);

	    if (PREDICT_FALSE (c0 == 0))
		{
		  vlib_node_increment_counter
		    (vm, node->node_index,
		     LI_HIT_ERROR_BUFFER_ALLOCATION_FAILURE, 1);
		  goto skip;
		}

        // 移动指针到udp头部
	    vlib_buffer_advance (c0, -sizeof (*iu0));
        
        // udp头
	    iu0 = vlib_buffer_get_current (c0);
        // ip头
	    ip0 = &iu0->ip4;

        // 修改ip头长度，ttl和协议
	    ip0->ip_version_and_header_length = 0x45;
	    ip0->ttl = 254;
	    ip0->protocol = IP_PROTOCOL_UDP;

        // 修改ip头源ip，目的ip，长度，校验和
	    ip0->src_address.as_u32 = lm->src_addrs[0].as_u32;
	    ip0->dst_address.as_u32 = lm->collectors[0].as_u32;
	    ip0->length = vlib_buffer_length_in_chain (vm, c0);
	    ip0->checksum = ip4_header_checksum (ip0);

        // 修改udp头源端口和目的端口都为配置的collector端口
	    udp0 = &iu0->udp;
	    udp0->src_port = udp0->dst_port = clib_host_to_net_u16 (lm->ports[0]);
	    udp0->checksum = 0;
	    udp0->length = clib_net_to_host_u16 (vlib_buffer_length_in_chain (vm, b0));

	    to_int_next[0] = vlib_get_buffer_index (vm, c0);
	    to_int_next++;
	  }

	skip:
	  if (PREDICT_FALSE ((node->flags & VLIB_NODE_FLAG_TRACE)
			     && (b0->flags & VLIB_BUFFER_IS_TRACED)))
	  {
	    li_hit_trace_t *t = vlib_add_trace (vm, node, b0, sizeof (*t));
	    t->next_index = next0;
	  }

	  /* verify speculative enqueue, maybe switch current next frame */
	  vlib_validate_buffer_enqueue_x1 (vm, node, next_index,
					   to_next, n_left_to_next,
					   bi0, next0);
	}

    vlib_put_next_frame (vm, node, next_index, n_left_to_next);
  }

  if (int_frame)
  {
    // 修改后的frame进入路由查找，最后发送到collector
    int_frame->n_vectors = frame->n_vectors;
    vlib_put_frame_to_node (vm, ip4_lookup_node.index, int_frame);
  }

  vlib_node_increment_counter (vm, li_hit_node.index,
			       LI_HIT_ERROR_HITS, frame->n_vectors);
  return frame->n_vectors;
}
```