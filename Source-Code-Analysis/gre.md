### GRE

### 配置

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
