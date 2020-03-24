### GRE

### GRE TEB和ERSPAN类型

#### vnet/gre/gre.c

```
/* *INDENT-OFF* */
VLIB_REGISTER_NODE (gre_encap_node) =
{
  .name = "gre-encap",
  .vector_size = sizeof (u32),
  .format_trace = format_gre_tx_trace,
  .type = VLIB_NODE_TYPE_INTERNAL,
  .n_errors = GRE_N_ERROR,
  .error_strings = gre_error_strings,
  .n_next_nodes = GRE_ENCAP_N_NEXT,
  .next_nodes = {
    [GRE_ENCAP_NEXT_L2_MIDCHAIN] = "adj-l2-midchain",
  },
};
/* *INDENT-ON* */
```

```
/**
 * @brief TX function. Only called for L2 payload including TEB or ERSPAN.
 *        L3 traffic uses the adj-midchains.
 */
VLIB_NODE_FN (gre_encap_node) (vlib_main_t * vm, vlib_node_runtime_t * node,
			       vlib_frame_t * frame)
{
  gre_main_t *gm = &gre_main;
  u32 *from, n_left_from;
  vlib_buffer_t *bufs[VLIB_FRAME_SIZE], **b = bufs;
  u32 sw_if_index[2] = { ~0, ~0 };
  const gre_tunnel_t *gt[2] = { 0 };
  adj_index_t adj_index[2] = { ADJ_INDEX_INVALID, ADJ_INDEX_INVALID };

  from = vlib_frame_vector_args (frame);
  n_left_from = frame->n_vectors;
  /* 将frame中的所有数据包buffer索引转换为指针 */
  vlib_get_buffers (vm, from, bufs, n_left_from);

  while (n_left_from >= 2)
    {

		if (PREDICT_FALSE
		  (sw_if_index[0] != vnet_buffer (b[0])->sw_if_index[VLIB_TX]))
			{
			  const vnet_hw_interface_t *hi;
			  sw_if_index[0] = vnet_buffer (b[0])->sw_if_index[VLIB_TX];
			  /* 获取抽象的gre类型的硬件接口 */
			  hi = vnet_get_sup_hw_interface (gm->vnet_main, sw_if_index[0]);
			  /* 获取gre隧道实例 */
			  gt[0] = &gm->tunnels[hi->dev_instance];
			  /* 获取gre隧道实例指向的midchain类型的adjacency索引 */
			  adj_index[0] = gt[0]->l2_adj_index;
			}
			  if (PREDICT_FALSE
			  (sw_if_index[1] != vnet_buffer (b[1])->sw_if_index[VLIB_TX]))
			{
			  const vnet_hw_interface_t *hi;
			  sw_if_index[1] = vnet_buffer (b[1])->sw_if_index[VLIB_TX];
			  hi = vnet_get_sup_hw_interface (gm->vnet_main, sw_if_index[1]);
			  gt[1] = &gm->tunnels[hi->dev_instance];
			  adj_index[1] = gt[1]->l2_adj_index;
			}

		    /* 给数据包中存储的adj索引赋值 */
		    vnet_buffer (b[0])->ip.adj_index[VLIB_TX] = adj_index[0];
		    vnet_buffer (b[1])->ip.adj_index[VLIB_TX] = adj_index[1];

			/ gre隧道类型为ERSPAN */
		    if (PREDICT_FALSE (gt[0]->type == GRE_TUNNEL_TYPE_ERSPAN))
			{
			  /* ERSPAN type II类型 */
			  /* Encap GRE seq# and ERSPAN type II header */
			  /* ERSPAN type II头部 */
			  erspan_t2_t *h0;
			  /* ERSPAN type II序列号  */
			  u32 seq_num;
			  u64 hdr;
			  /* 移动current_data指针到ERSPAN头部开始位置 */
			  vlib_buffer_advance (b[0], -sizeof (erspan_t2_t));
			  /* 获取ERSPAN头部 */
			  h0 = vlib_buffer_get_current (b[0]);
			  /* ERSPAN type II序列号赋值，为gre_sn->seq_num每次+1，线程安全 */
			  seq_num = clib_atomic_fetch_add (&gt[0]->gre_sn->seq_num, 1);
			  /*  ERSPAN type II头部赋初值 */
			  hdr = clib_host_to_net_u64 (ERSPAN_HDR2);
			  /*  ERSPAN type II seq num赋值 */
			  h0->seq_num = clib_host_to_net_u32 (seq_num);
			  h0->t2_u64 = hdr;
			  /* ERSPAN type II session id赋值 */
			  h0->t2.cos_en_t_session |= clib_host_to_net_u16 (gt[0]->session_id);
			}
		    if (PREDICT_FALSE (gt[1]->type == GRE_TUNNEL_TYPE_ERSPAN))
			{
			  /* Encap GRE seq# and ERSPAN type II header */
			  erspan_t2_t *h0;
			  u32 seq_num;
			  u64 hdr;
			  vlib_buffer_advance (b[1], -sizeof (erspan_t2_t));
			  h0 = vlib_buffer_get_current (b[1]);
			  seq_num = clib_atomic_fetch_add (&gt[1]->gre_sn->seq_num, 1);
			  hdr = clib_host_to_net_u64 (ERSPAN_HDR2);
			  h0->seq_num = clib_host_to_net_u32 (seq_num);
			  h0->t2_u64 = hdr;
			  h0->t2.cos_en_t_session |= clib_host_to_net_u16 (gt[1]->session_id);
			}

			/* 检查数据包b0的trace标记 */
		    if (PREDICT_FALSE (b[0]->flags & VLIB_BUFFER_IS_TRACED))
			{
			  /* 收集trace信息 */
			  gre_tx_trace_t *tr = vlib_add_trace (vm, node,
								   b[0], sizeof (*tr));
			  tr->tunnel_id = gt[0] - gm->tunnels;
			  tr->src = gt[0]->tunnel_src;
			  tr->dst = gt[0]->tunnel_dst.fp_addr;
			  tr->length = vlib_buffer_length_in_chain (vm, b[0]);
			}
		    if (PREDICT_FALSE (b[1]->flags & VLIB_BUFFER_IS_TRACED))
			{
			  gre_tx_trace_t *tr = vlib_add_trace (vm, node,
								   b[1], sizeof (*tr));
			  tr->tunnel_id = gt[1] - gm->tunnels;
			  tr->src = gt[1]->tunnel_src;
			  tr->dst = gt[1]->tunnel_dst.fp_addr;
			  tr->length = vlib_buffer_length_in_chain (vm, b[1]);
			}

        b += 2;
        n_left_from -= 2;
    }

  while (n_left_from >= 1)
    {

      if (PREDICT_FALSE
	  (sw_if_index[0] != vnet_buffer (b[0])->sw_if_index[VLIB_TX]))
		{
		  const vnet_hw_interface_t *hi;
		  sw_if_index[0] = vnet_buffer (b[0])->sw_if_index[VLIB_TX];
		  hi = vnet_get_sup_hw_interface (gm->vnet_main, sw_if_index[0]);
		  gt[0] = &gm->tunnels[hi->dev_instance];
		  adj_index[0] = gt[0]->l2_adj_index;
		}

		  vnet_buffer (b[0])->ip.adj_index[VLIB_TX] = adj_index[0];

		  if (PREDICT_FALSE (gt[0]->type == GRE_TUNNEL_TYPE_ERSPAN))
		{
		  /* Encap GRE seq# and ERSPAN type II header */
		  erspan_t2_t *h0;
		  u32 seq_num;
		  u64 hdr;
		  vlib_buffer_advance (b[0], -sizeof (erspan_t2_t));
		  h0 = vlib_buffer_get_current (b[0]);
		  seq_num = clib_atomic_fetch_add (&gt[0]->gre_sn->seq_num, 1);
		  hdr = clib_host_to_net_u64 (ERSPAN_HDR2);
		  h0->seq_num = clib_host_to_net_u32 (seq_num);
		  h0->t2_u64 = hdr;
		  h0->t2.cos_en_t_session |= clib_host_to_net_u16 (gt[0]->session_id);
		}

		  if (PREDICT_FALSE (b[0]->flags & VLIB_BUFFER_IS_TRACED))
		{
		  gre_tx_trace_t *tr = vlib_add_trace (vm, node,
							   b[0], sizeof (*tr));
		  tr->tunnel_id = gt[0] - gm->tunnels;
		  tr->src = gt[0]->tunnel_src;
		  tr->dst = gt[0]->tunnel_dst.fp_addr;
		  tr->length = vlib_buffer_length_in_chain (vm, b[0]);
		}

      b += 1;
      n_left_from -= 1;
    }

  /* 把frame传到下个node（adj-l2-midchain）处理 */
  vlib_buffer_enqueue_to_single_next (vm, node, from,
				      GRE_ENCAP_NEXT_L2_MIDCHAIN,
				      frame->n_vectors);
  /* 增加GRE encap计数器 */
  vlib_node_increment_counter (vm, node->node_index,
			       GRE_ERROR_PKTS_ENCAP, frame->n_vectors);

  return frame->n_vectors;
}
```

### GRE处理

#### vnet/gre/node.c

```
/* *INDENT-OFF* */
VLIB_REGISTER_NODE (gre4_input_node) = {
  .name = "gre4-input",
  /* Takes a vector of packets. */
  .vector_size = sizeof (u32),

  .n_errors = GRE_N_ERROR,
  .error_strings = gre_error_strings,

  .n_next_nodes = GRE_INPUT_N_NEXT,
  .next_nodes = {
#define _(s,n) [GRE_INPUT_NEXT_##s] = n,
    foreach_gre_input_next
#undef _
  },

  .format_buffer = format_gre_header_with_length,
  .format_trace = format_gre_rx_trace,
  .unformat_buffer = unformat_gre_header,
};

VLIB_REGISTER_NODE (gre6_input_node) = {
  .name = "gre6-input",
  /* Takes a vector of packets. */
  .vector_size = sizeof (u32),

  .runtime_data_bytes = sizeof (gre_input_runtime_t),

  .n_errors = GRE_N_ERROR,
  .error_strings = gre_error_strings,

  .n_next_nodes = GRE_INPUT_N_NEXT,
  .next_nodes = {
#define _(s,n) [GRE_INPUT_NEXT_##s] = n,
    foreach_gre_input_next
#undef _
  },

  .format_buffer = format_gre_header_with_length,
  .format_trace = format_gre_rx_trace,
  .unformat_buffer = unformat_gre_header,
};
/* *INDENT-ON* */
```

```
always_inline uword
gre_input (vlib_main_t * vm,
	   vlib_node_runtime_t * node, vlib_frame_t * frame,
	   const int is_ipv6)
{
  gre_main_t *gm = &gre_main;
  u32 *from, n_left_from;
  vlib_buffer_t *bufs[VLIB_FRAME_SIZE], **b = bufs;
  u16 nexts[VLIB_FRAME_SIZE], *next = nexts;
  u16 cached_protocol = ~0;
  u32 cached_next_index = SPARSE_VEC_INVALID_INDEX;
  u32 cached_tun_sw_if_index = ~0;
  /* gre头部key字段，有src和dst ip生成，隧道接收端用此key进行校验 */
  gre_tunnel_key_t cached_key;

  from = vlib_frame_vector_args (frame);
  n_left_from = frame->n_vectors;
  vlib_get_buffers (vm, from, bufs, n_left_from);

  /* 初始化gre key字段 */
  if (is_ipv6)
    clib_memset (&cached_key.gtk_v6, 0xff, sizeof (cached_key.gtk_v6));
  else
    clib_memset (&cached_key.gtk_v4, 0xff, sizeof (cached_key.gtk_v4));

  while (n_left_from >= 2)
    {
	  /* 用于传输的外部ip头部 */
      const ip6_header_t *ip6[2];
      const ip4_header_t *ip4[2];
	  /* 用于封装的gre头部 */
      const gre_header_t *gre[2];
      u32 nidx[2];
      next_info_t ni[2];
      u8 type[2];
      u16 version[2];
      u32 len[2];
      gre_tunnel_key_t key[2];
      u8 matched[2];
      u32 tun_sw_if_index[2];

	  /* 预取后4个数据包 */
      if (PREDICT_TRUE (n_left_from >= 6))
	{
	  vlib_prefetch_buffer_data (b[2], LOAD);
	  vlib_prefetch_buffer_data (b[3], LOAD);
	  vlib_prefetch_buffer_header (b[4], STORE);
	  vlib_prefetch_buffer_header (b[5], STORE);
	}

      if (is_ipv6)
	{
	  /* ip6_local hands us the ip header, not the gre header */
	  ip6[0] = vlib_buffer_get_current (b[0]);
	  ip6[1] = vlib_buffer_get_current (b[1]);
	  gre[0] = (void *) (ip6[0] + 1);
	  gre[1] = (void *) (ip6[1] + 1);
	  vlib_buffer_advance (b[0], sizeof (*ip6[0]) + sizeof (*gre[0]));
	  vlib_buffer_advance (b[1], sizeof (*ip6[0]) + sizeof (*gre[0]));
	}
      else
	{
	  /* ip4_local hands us the ip header, not the gre header */
	  /* 设置外部ip头指针，待填充 */
	  ip4[0] = vlib_buffer_get_current (b[0]);
	  ip4[1] = vlib_buffer_get_current (b[1]);
	  /* 设置gre头指针，待填充 */
	  gre[0] = (void *) (ip4[0] + 1);
	  gre[1] = (void *) (ip4[1] + 1);
	  /*  */
	  vlib_buffer_advance (b[0], sizeof (*ip4[0]) + sizeof (*gre[0]));
	  vlib_buffer_advance (b[1], sizeof (*ip4[0]) + sizeof (*gre[0]));
	}
	
	  /* 根据gre头部上层乘客协议字段，获取next节点索引 */
      if (PREDICT_TRUE (cached_protocol == gre[0]->protocol))
	{
	  nidx[0] = cached_next_index;
	}
      else
	{
	  cached_next_index = nidx[0] =
	    sparse_vec_index (gm->next_by_protocol, gre[0]->protocol);
	  cached_protocol = gre[0]->protocol;
	}
      if (PREDICT_TRUE (cached_protocol == gre[1]->protocol))
	{
	  nidx[1] = cached_next_index;
	}
      else
	{
	  cached_next_index = nidx[1] =
	    sparse_vec_index (gm->next_by_protocol, gre[1]->protocol);
	  cached_protocol = gre[1]->protocol;
	}

	  /*  */
      ni[0] = vec_elt (gm->next_by_protocol, nidx[0]);
      ni[1] = vec_elt (gm->next_by_protocol, nidx[1]);
      next[0] = ni[0].next_index;
      next[1] = ni[1].next_index;
      type[0] = ni[0].tunnel_type;
      type[1] = ni[1].tunnel_type;

      b[0]->error = nidx[0] == SPARSE_VEC_INVALID_INDEX
	? node->errors[GRE_ERROR_UNKNOWN_PROTOCOL]
	: node->errors[GRE_ERROR_NONE];
      b[1]->error = nidx[1] == SPARSE_VEC_INVALID_INDEX
	? node->errors[GRE_ERROR_UNKNOWN_PROTOCOL]
	: node->errors[GRE_ERROR_NONE];

	  /* 获取gre头flags_and_version字段(共1 byte) */
      version[0] = clib_net_to_host_u16 (gre[0]->flags_and_version);
      version[1] = clib_net_to_host_u16 (gre[1]->flags_and_version);
	  /* version为flags_and_version最后3bits */
      version[0] &= GRE_VERSION_MASK;
      version[1] &= GRE_VERSION_MASK;

	  /* 检查gre版本，版本1为PPTP(RFC2637)，不支持 */
      b[0]->error = version[0]
	? node->errors[GRE_ERROR_UNSUPPORTED_VERSION] : b[0]->error;
      next[0] = version[0] ? GRE_INPUT_NEXT_DROP : next[0];
      b[1]->error = version[1]
	? node->errors[GRE_ERROR_UNSUPPORTED_VERSION] : b[1]->error;
      next[1] = version[1] ? GRE_INPUT_NEXT_DROP : next[1];

	  /* 获取buffer链中所有buffer的总长度 */
      len[0] = vlib_buffer_length_in_chain (vm, b[0]);
      len[1] = vlib_buffer_length_in_chain (vm, b[1]);

      /* always search for P2P types in the DP */
      if (is_ipv6)
	{
	  gre_mk_key6 (&ip6[0]->dst_address,
		       &ip6[0]->src_address,
		       vnet_buffer (b[0])->ip.fib_index,
		       type[0], TUNNEL_MODE_P2P, 0, &key[0].gtk_v6);
	  gre_mk_key6 (&ip6[1]->dst_address,
		       &ip6[1]->src_address,
		       vnet_buffer (b[1])->ip.fib_index,
		       type[1], TUNNEL_MODE_P2P, 0, &key[1].gtk_v6);
	  matched[0] = gre_match_key6 (&cached_key.gtk_v6, &key[0].gtk_v6);
	  matched[1] = gre_match_key6 (&cached_key.gtk_v6, &key[1].gtk_v6);
	}
      else
	{
	  /* 生成gre的key字段 */
	  gre_mk_key4 (ip4[0]->dst_address,
		       ip4[0]->src_address,
		       vnet_buffer (b[0])->ip.fib_index,
		       type[0], TUNNEL_MODE_P2P, 0, &key[0].gtk_v4);
	  gre_mk_key4 (ip4[1]->dst_address,
		       ip4[1]->src_address,
		       vnet_buffer (b[1])->ip.fib_index,
		       type[1], TUNNEL_MODE_P2P, 0, &key[1].gtk_v4);
	  matched[0] = gre_match_key4 (&cached_key.gtk_v4, &key[0].gtk_v4);
	  matched[1] = gre_match_key4 (&cached_key.gtk_v4, &key[1].gtk_v4);
	}

      tun_sw_if_index[0] = cached_tun_sw_if_index;
      tun_sw_if_index[1] = cached_tun_sw_if_index;
      if (PREDICT_FALSE (!matched[0]))
	gre_tunnel_get (gm, node, b[0], &next[0], &key[0], &cached_key,
			&tun_sw_if_index[0], &cached_tun_sw_if_index,
			is_ipv6);
      if (PREDICT_FALSE (!matched[1]))
	gre_tunnel_get (gm, node, b[1], &next[1], &key[1], &cached_key,
			&tun_sw_if_index[1], &cached_tun_sw_if_index,
			is_ipv6);

      if (PREDICT_TRUE (next[0] > GRE_INPUT_NEXT_DROP))
	{
	  vlib_increment_combined_counter (&gm->vnet_main->
					   interface_main.combined_sw_if_counters
					   [VNET_INTERFACE_COUNTER_RX],
					   vm->thread_index,
					   tun_sw_if_index[0],
					   1 /* packets */ ,
					   len[0] /* bytes */ );
	  vnet_buffer (b[0])->sw_if_index[VLIB_RX] = tun_sw_if_index[0];
	}
      if (PREDICT_TRUE (next[1] > GRE_INPUT_NEXT_DROP))
	{
	  vlib_increment_combined_counter (&gm->vnet_main->
					   interface_main.combined_sw_if_counters
					   [VNET_INTERFACE_COUNTER_RX],
					   vm->thread_index,
					   tun_sw_if_index[1],
					   1 /* packets */ ,
					   len[1] /* bytes */ );
	  vnet_buffer (b[1])->sw_if_index[VLIB_RX] = tun_sw_if_index[1];
	}

      if (PREDICT_FALSE (b[0]->flags & VLIB_BUFFER_IS_TRACED))
	gre_trace (vm, node, b[0], tun_sw_if_index[0], ip6[0], ip4[0],
		   is_ipv6);
      if (PREDICT_FALSE (b[1]->flags & VLIB_BUFFER_IS_TRACED))
	gre_trace (vm, node, b[1], tun_sw_if_index[1], ip6[1], ip4[1],
		   is_ipv6);

      b += 2;
      next += 2;
      n_left_from -= 2;
    }

  while (n_left_from >= 1)
    {
      const ip6_header_t *ip6[1];
      const ip4_header_t *ip4[1];
      const gre_header_t *gre[1];
      u32 nidx[1];
      next_info_t ni[1];
      u8 type[1];
      u16 version[1];
      u32 len[1];
      gre_tunnel_key_t key[1];
      u8 matched[1];
      u32 tun_sw_if_index[1];

      if (PREDICT_TRUE (n_left_from >= 3))
	{
	  vlib_prefetch_buffer_data (b[1], LOAD);
	  vlib_prefetch_buffer_header (b[2], STORE);
	}

      if (is_ipv6)
	{
	  /* ip6_local hands us the ip header, not the gre header */
	  ip6[0] = vlib_buffer_get_current (b[0]);
	  gre[0] = (void *) (ip6[0] + 1);
	  vlib_buffer_advance (b[0], sizeof (*ip6[0]) + sizeof (*gre[0]));
	}
      else
	{
	  /* ip4_local hands us the ip header, not the gre header */
	  ip4[0] = vlib_buffer_get_current (b[0]);
	  gre[0] = (void *) (ip4[0] + 1);
	  vlib_buffer_advance (b[0], sizeof (*ip4[0]) + sizeof (*gre[0]));
	}

      if (PREDICT_TRUE (cached_protocol == gre[0]->protocol))
	{
	  nidx[0] = cached_next_index;
	}
      else
	{
	  cached_next_index = nidx[0] =
	    sparse_vec_index (gm->next_by_protocol, gre[0]->protocol);
	  cached_protocol = gre[0]->protocol;
	}

      ni[0] = vec_elt (gm->next_by_protocol, nidx[0]);
      next[0] = ni[0].next_index;
      type[0] = ni[0].tunnel_type;

      b[0]->error = nidx[0] == SPARSE_VEC_INVALID_INDEX
	? node->errors[GRE_ERROR_UNKNOWN_PROTOCOL]
	: node->errors[GRE_ERROR_NONE];

      version[0] = clib_net_to_host_u16 (gre[0]->flags_and_version);
      version[0] &= GRE_VERSION_MASK;

      b[0]->error = version[0]
	? node->errors[GRE_ERROR_UNSUPPORTED_VERSION] : b[0]->error;
      next[0] = version[0] ? GRE_INPUT_NEXT_DROP : next[0];

      len[0] = vlib_buffer_length_in_chain (vm, b[0]);

      if (is_ipv6)
	{
	  gre_mk_key6 (&ip6[0]->dst_address,
		       &ip6[0]->src_address,
		       vnet_buffer (b[0])->ip.fib_index,
		       type[0], TUNNEL_MODE_P2P, 0, &key[0].gtk_v6);
	  matched[0] = gre_match_key6 (&cached_key.gtk_v6, &key[0].gtk_v6);
	}
      else
	{
	  gre_mk_key4 (ip4[0]->dst_address,
		       ip4[0]->src_address,
		       vnet_buffer (b[0])->ip.fib_index,
		       type[0], TUNNEL_MODE_P2P, 0, &key[0].gtk_v4);
	  matched[0] = gre_match_key4 (&cached_key.gtk_v4, &key[0].gtk_v4);
	}

      tun_sw_if_index[0] = cached_tun_sw_if_index;
      if (PREDICT_FALSE (!matched[0]))
	gre_tunnel_get (gm, node, b[0], &next[0], &key[0], &cached_key,
			&tun_sw_if_index[0], &cached_tun_sw_if_index,
			is_ipv6);

      if (PREDICT_TRUE (next[0] > GRE_INPUT_NEXT_DROP))
	{
	  vlib_increment_combined_counter (&gm->vnet_main->
					   interface_main.combined_sw_if_counters
					   [VNET_INTERFACE_COUNTER_RX],
					   vm->thread_index,
					   tun_sw_if_index[0],
					   1 /* packets */ ,
					   len[0] /* bytes */ );
	  vnet_buffer (b[0])->sw_if_index[VLIB_RX] = tun_sw_if_index[0];
	}

      if (PREDICT_FALSE (b[0]->flags & VLIB_BUFFER_IS_TRACED))
	gre_trace (vm, node, b[0], tun_sw_if_index[0], ip6[0], ip4[0],
		   is_ipv6);

      b += 1;
      next += 1;
      n_left_from -= 1;
    }

  vlib_buffer_enqueue_to_next (vm, node, from, nexts, frame->n_vectors);

  vlib_node_increment_counter (vm,
			       is_ipv6 ? gre6_input_node.index :
			       gre4_input_node.index, GRE_ERROR_PKTS_DECAP,
			       n_left_from);

  return frame->n_vectors;
}
```
