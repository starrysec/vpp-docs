## ip

重点分析ip4 forward，源码目录`vnet/ip/ip_forward.c`。

### ip4-lookup

```
VLIB_REGISTER_NODE (ip4_lookup_node) =
{
  .name = "ip4-lookup",
  .vector_size = sizeof (u32),
  .format_trace = format_ip4_lookup_trace,
  .n_next_nodes = IP_LOOKUP_N_NEXT,
  .next_nodes = IP4_LOOKUP_NEXT_NODES,
};
```

```
/** @brief IPv4 lookup node.
    @node ip4-lookup

    This is the main IPv4 lookup dispatch node.

    @param vm vlib_main_t corresponding to the current thread
    @param node vlib_node_runtime_t
    @param frame vlib_frame_t whose contents should be dispatched

    @par Graph mechanics: buffer metadata, next index usage

    @em Uses:
    - <code>vnet_buffer(b)->sw_if_index[VLIB_RX]</code>
        - Indicates the @c sw_if_index value of the interface that the
	  packet was received on.
    - <code>vnet_buffer(b)->sw_if_index[VLIB_TX]</code>
        - When the value is @c ~0 then the node performs a longest prefix
          match (LPM) for the packet destination address in the FIB attached
          to the receive interface.
        - Otherwise perform LPM for the packet destination address in the
          indicated FIB. In this case <code>[VLIB_TX]</code> is a FIB index
          value (0, 1, ...) and not a VRF id.

    @em Sets:
    - <code>vnet_buffer(b)->ip.adj_index[VLIB_TX]</code>
        - The lookup result adjacency index.

    <em>Next Index:</em>
    - Dispatches the packet to the node index found in
      ip_adjacency_t @c adj->lookup_next_index
      (where @c adj is the lookup result adjacency).
*/
VLIB_NODE_FN (ip4_lookup_node) (vlib_main_t * vm, vlib_node_runtime_t * node,
				vlib_frame_t * frame)
{
  return ip4_lookup_inline (vm, node, frame);
}
```

```
/**
 * @file
 * @brief IPv4 Forwarding.
 *
 * This file contains the source code for IPv4 forwarding.
 */

always_inline uword
ip4_lookup_inline (vlib_main_t * vm,
		   vlib_node_runtime_t * node, vlib_frame_t * frame)
{
  ip4_main_t *im = &ip4_main;
  vlib_combined_counter_main_t *cm = &load_balance_main.lbm_to_counters;
  u32 n_left, *from;
  u32 thread_index = vm->thread_index;
  vlib_buffer_t *bufs[VLIB_FRAME_SIZE];
  vlib_buffer_t **b = bufs;
  u16 nexts[VLIB_FRAME_SIZE], *next;

  from = vlib_frame_vector_args (frame);
  n_left = frame->n_vectors;
  next = nexts;
  vlib_get_buffers (vm, from, bufs, n_left);

#if (CLIB_N_PREFETCHES >= 8)
  while (n_left >= 4)
  {
    ip4_header_t *ip0, *ip1, *ip2, *ip3;
    const load_balance_t *lb0, *lb1, *lb2, *lb3;
    ip4_fib_mtrie_t *mtrie0, *mtrie1, *mtrie2, *mtrie3;
    ip4_fib_mtrie_leaf_t leaf0, leaf1, leaf2, leaf3;
    ip4_address_t *dst_addr0, *dst_addr1, *dst_addr2, *dst_addr3;
    u32 lb_index0, lb_index1, lb_index2, lb_index3;
    flow_hash_config_t flow_hash_config0, flow_hash_config1;
    flow_hash_config_t flow_hash_config2, flow_hash_config3;
    u32 hash_c0, hash_c1, hash_c2, hash_c3;
    const dpo_id_t *dpo0, *dpo1, *dpo2, *dpo3;

    /* Prefetch next iteration. */
    if (n_left >= 8)
	{
	  vlib_prefetch_buffer_header (b[4], LOAD);
	  vlib_prefetch_buffer_header (b[5], LOAD);
	  vlib_prefetch_buffer_header (b[6], LOAD);
	  vlib_prefetch_buffer_header (b[7], LOAD);

	  CLIB_PREFETCH (b[4]->data, sizeof (ip0[0]), LOAD);
	  CLIB_PREFETCH (b[5]->data, sizeof (ip0[0]), LOAD);
	  CLIB_PREFETCH (b[6]->data, sizeof (ip0[0]), LOAD);
	  CLIB_PREFETCH (b[7]->data, sizeof (ip0[0]), LOAD);
	}

    ip0 = vlib_buffer_get_current (b[0]);
    ip1 = vlib_buffer_get_current (b[1]);
    ip2 = vlib_buffer_get_current (b[2]);
    ip3 = vlib_buffer_get_current (b[3]);

    // 数据包目的地址
    dst_addr0 = &ip0->dst_address;
    dst_addr1 = &ip1->dst_address;
    dst_addr2 = &ip2->dst_address;
    dst_addr3 = &ip3->dst_address;

    // 设置数据包lookup用的fib索引：
    // 如果(b)->sw_if_index[VLIB_TX]未设置则fib索引为(b)->sw_if_index[VLIB_RX]
    // 否则fib索引为(b)->sw_if_index[VLIB_TX]
    ip_lookup_set_buffer_fib_index (im->fib_index_by_sw_if_index, b[0]);
    ip_lookup_set_buffer_fib_index (im->fib_index_by_sw_if_index, b[1]);
    ip_lookup_set_buffer_fib_index (im->fib_index_by_sw_if_index, b[2]);
    ip_lookup_set_buffer_fib_index (im->fib_index_by_sw_if_index, b[3]);

    // 根据fib索引，获取FIB(mtrie树)
    mtrie0 = &ip4_fib_get (vnet_buffer (b[0])->ip.fib_index)->mtrie;
    mtrie1 = &ip4_fib_get (vnet_buffer (b[1])->ip.fib_index)->mtrie;
    mtrie2 = &ip4_fib_get (vnet_buffer (b[2])->ip.fib_index)->mtrie;
    mtrie3 = &ip4_fib_get (vnet_buffer (b[3])->ip.fib_index)->mtrie;

    // 根据数据包目的地址，在FIB中查找叶子节点
    leaf0 = ip4_fib_mtrie_lookup_step_one (mtrie0, dst_addr0);
    leaf1 = ip4_fib_mtrie_lookup_step_one (mtrie1, dst_addr1);
    leaf2 = ip4_fib_mtrie_lookup_step_one (mtrie2, dst_addr2);
    leaf3 = ip4_fib_mtrie_lookup_step_one (mtrie3, dst_addr3);

    leaf0 = ip4_fib_mtrie_lookup_step (mtrie0, leaf0, dst_addr0, 2);
    leaf1 = ip4_fib_mtrie_lookup_step (mtrie1, leaf1, dst_addr1, 2);
    leaf2 = ip4_fib_mtrie_lookup_step (mtrie2, leaf2, dst_addr2, 2);
    leaf3 = ip4_fib_mtrie_lookup_step (mtrie3, leaf3, dst_addr3, 2);

    leaf0 = ip4_fib_mtrie_lookup_step (mtrie0, leaf0, dst_addr0, 3);
    leaf1 = ip4_fib_mtrie_lookup_step (mtrie1, leaf1, dst_addr1, 3);
    leaf2 = ip4_fib_mtrie_lookup_step (mtrie2, leaf2, dst_addr2, 3);
    leaf3 = ip4_fib_mtrie_lookup_step (mtrie3, leaf3, dst_addr3, 3);

    // 根据叶子节点，查找adj索引
    lb_index0 = ip4_fib_mtrie_leaf_get_adj_index (leaf0);
    lb_index1 = ip4_fib_mtrie_leaf_get_adj_index (leaf1);
    lb_index2 = ip4_fib_mtrie_leaf_get_adj_index (leaf2);
    lb_index3 = ip4_fib_mtrie_leaf_get_adj_index (leaf3);

    ASSERT (lb_index0 && lb_index1 && lb_index2 && lb_index3);
    // 根据adj索引获取lb（load_balance_t）结构（The encapsulation breakages are for fast DP access）
    lb0 = load_balance_get (lb_index0);
    lb1 = load_balance_get (lb_index1);
    lb2 = load_balance_get (lb_index2);
    lb3 = load_balance_get (lb_index3);

    /* 检测lb中用于转发的桶数，桶数是2的指数 */
    ASSERT (lb0->lb_n_buckets > 0);
    ASSERT (is_pow2 (lb0->lb_n_buckets));
    ASSERT (lb1->lb_n_buckets > 0);
    ASSERT (is_pow2 (lb1->lb_n_buckets));
    ASSERT (lb2->lb_n_buckets > 0);
    ASSERT (is_pow2 (lb2->lb_n_buckets));
    ASSERT (lb3->lb_n_buckets > 0);
    ASSERT (is_pow2 (lb3->lb_n_buckets));

    /* Use flow hash to compute multipath adjacency. */
    /* 初始化数据包流hash为0 */
    hash_c0 = vnet_buffer (b[0])->ip.flow_hash = 0;
    hash_c1 = vnet_buffer (b[1])->ip.flow_hash = 0;
    hash_c2 = vnet_buffer (b[2])->ip.flow_hash = 0;
    hash_c3 = vnet_buffer (b[3])->ip.flow_hash = 0;
    /* lb中没有转发桶 */ 
    if (PREDICT_FALSE (lb0->lb_n_buckets > 1))
	{
      /* 计算流hash */
	  flow_hash_config0 = lb0->lb_hash_config;
	  hash_c0 = vnet_buffer (b[0])->ip.flow_hash =
	    ip4_compute_flow_hash (ip0, flow_hash_config0);
      /* 根据流hash，获取用于转发的桶，即dpo */
	  dpo0 =
	    load_balance_get_fwd_bucket (lb0,
					 (hash_c0 &
					  (lb0->lb_n_buckets_minus_1)));
	}
    /* lb中有转发桶 */
    else
	{
      /* 获取第一个转发桶，即dpo */
	  dpo0 = load_balance_get_bucket_i (lb0, 0);
	}
    if (PREDICT_FALSE (lb1->lb_n_buckets > 1))
	{
	  flow_hash_config1 = lb1->lb_hash_config;
	  hash_c1 = vnet_buffer (b[1])->ip.flow_hash =
	    ip4_compute_flow_hash (ip1, flow_hash_config1);
	  dpo1 =
	    load_balance_get_fwd_bucket (lb1,
					 (hash_c1 &
					  (lb1->lb_n_buckets_minus_1)));
	}
    else
	{
	  dpo1 = load_balance_get_bucket_i (lb1, 0);
	}
    if (PREDICT_FALSE (lb2->lb_n_buckets > 1))
	{
	  flow_hash_config2 = lb2->lb_hash_config;
	  hash_c2 = vnet_buffer (b[2])->ip.flow_hash =
	    ip4_compute_flow_hash (ip2, flow_hash_config2);
	  dpo2 =
	    load_balance_get_fwd_bucket (lb2,
					 (hash_c2 &
					  (lb2->lb_n_buckets_minus_1)));
	}
    else
	{
	  dpo2 = load_balance_get_bucket_i (lb2, 0);
	}
    if (PREDICT_FALSE (lb3->lb_n_buckets > 1))
	{
	  flow_hash_config3 = lb3->lb_hash_config;
	  hash_c3 = vnet_buffer (b[3])->ip.flow_hash =
	    ip4_compute_flow_hash (ip3, flow_hash_config3);
	  dpo3 =
	    load_balance_get_fwd_bucket (lb3,
					 (hash_c3 &
					  (lb3->lb_n_buckets_minus_1)));
	}
    else
	{
	  dpo3 = load_balance_get_bucket_i (lb3, 0);
	}

    // 获取下个node节点索引
    next[0] = dpo0->dpoi_next_node;
    vnet_buffer (b[0])->ip.adj_index[VLIB_TX] = dpo0->dpoi_index;
    next[1] = dpo1->dpoi_next_node;
    vnet_buffer (b[1])->ip.adj_index[VLIB_TX] = dpo1->dpoi_index;
    next[2] = dpo2->dpoi_next_node;
    vnet_buffer (b[2])->ip.adj_index[VLIB_TX] = dpo2->dpoi_index;
    next[3] = dpo3->dpoi_next_node;
    vnet_buffer (b[3])->ip.adj_index[VLIB_TX] = dpo3->dpoi_index;

    // 更新线程的lb联计数器值，包括cpu index, packets，bytes等
    vlib_increment_combined_counter
	  (cm, thread_index, lb_index0, 1,
	  vlib_buffer_length_in_chain (vm, b[0]));
    vlib_increment_combined_counter
	  (cm, thread_index, lb_index1, 1,
	  vlib_buffer_length_in_chain (vm, b[1]));
    vlib_increment_combined_counter
	  (cm, thread_index, lb_index2, 1,
	  vlib_buffer_length_in_chain (vm, b[2]));
    vlib_increment_combined_counter
	  (cm, thread_index, lb_index3, 1,
	  vlib_buffer_length_in_chain (vm, b[3]));

    b += 4;
    next += 4;
    n_left -= 4;
  }
#elif (CLIB_N_PREFETCHES >= 4)
  while (n_left >= 4)
  {
    ip4_header_t *ip0, *ip1;
    const load_balance_t *lb0, *lb1;
    ip4_fib_mtrie_t *mtrie0, *mtrie1;
    ip4_fib_mtrie_leaf_t leaf0, leaf1;
    ip4_address_t *dst_addr0, *dst_addr1;
    u32 lb_index0, lb_index1;
    flow_hash_config_t flow_hash_config0, flow_hash_config1;
    u32 hash_c0, hash_c1;
    const dpo_id_t *dpo0, *dpo1;

    /* Prefetch next iteration. */
    {
	  vlib_prefetch_buffer_header (b[2], LOAD);
	  vlib_prefetch_buffer_header (b[3], LOAD);

	  CLIB_PREFETCH (b[2]->data, sizeof (ip0[0]), LOAD);
	  CLIB_PREFETCH (b[3]->data, sizeof (ip0[0]), LOAD);
    }

    ip0 = vlib_buffer_get_current (b[0]);
    ip1 = vlib_buffer_get_current (b[1]);

    dst_addr0 = &ip0->dst_address;
    dst_addr1 = &ip1->dst_address;

    ip_lookup_set_buffer_fib_index (im->fib_index_by_sw_if_index, b[0]);
    ip_lookup_set_buffer_fib_index (im->fib_index_by_sw_if_index, b[1]);

    mtrie0 = &ip4_fib_get (vnet_buffer (b[0])->ip.fib_index)->mtrie;
    mtrie1 = &ip4_fib_get (vnet_buffer (b[1])->ip.fib_index)->mtrie;

    leaf0 = ip4_fib_mtrie_lookup_step_one (mtrie0, dst_addr0);
    leaf1 = ip4_fib_mtrie_lookup_step_one (mtrie1, dst_addr1);

    leaf0 = ip4_fib_mtrie_lookup_step (mtrie0, leaf0, dst_addr0, 2);
    leaf1 = ip4_fib_mtrie_lookup_step (mtrie1, leaf1, dst_addr1, 2);

    leaf0 = ip4_fib_mtrie_lookup_step (mtrie0, leaf0, dst_addr0, 3);
    leaf1 = ip4_fib_mtrie_lookup_step (mtrie1, leaf1, dst_addr1, 3);

    lb_index0 = ip4_fib_mtrie_leaf_get_adj_index (leaf0);
    lb_index1 = ip4_fib_mtrie_leaf_get_adj_index (leaf1);

    ASSERT (lb_index0 && lb_index1);
    lb0 = load_balance_get (lb_index0);
    lb1 = load_balance_get (lb_index1);

    ASSERT (lb0->lb_n_buckets > 0);
    ASSERT (is_pow2 (lb0->lb_n_buckets));
    ASSERT (lb1->lb_n_buckets > 0);
    ASSERT (is_pow2 (lb1->lb_n_buckets));

    /* Use flow hash to compute multipath adjacency. */
    hash_c0 = vnet_buffer (b[0])->ip.flow_hash = 0;
    hash_c1 = vnet_buffer (b[1])->ip.flow_hash = 0;
    if (PREDICT_FALSE (lb0->lb_n_buckets > 1))
	{
	  flow_hash_config0 = lb0->lb_hash_config;
	  hash_c0 = vnet_buffer (b[0])->ip.flow_hash =
	    ip4_compute_flow_hash (ip0, flow_hash_config0);
	  dpo0 =
	    load_balance_get_fwd_bucket (lb0,
					 (hash_c0 &
					  (lb0->lb_n_buckets_minus_1)));
	}
    else
	{
	  dpo0 = load_balance_get_bucket_i (lb0, 0);
	}
    if (PREDICT_FALSE (lb1->lb_n_buckets > 1))
	{
	  flow_hash_config1 = lb1->lb_hash_config;
	  hash_c1 = vnet_buffer (b[1])->ip.flow_hash =
	    ip4_compute_flow_hash (ip1, flow_hash_config1);
	  dpo1 =
	    load_balance_get_fwd_bucket (lb1,
					 (hash_c1 &
					  (lb1->lb_n_buckets_minus_1)));
	}
    else
	{
	  dpo1 = load_balance_get_bucket_i (lb1, 0);
	}

    next[0] = dpo0->dpoi_next_node;
    vnet_buffer (b[0])->ip.adj_index[VLIB_TX] = dpo0->dpoi_index;
    next[1] = dpo1->dpoi_next_node;
    vnet_buffer (b[1])->ip.adj_index[VLIB_TX] = dpo1->dpoi_index;

    vlib_increment_combined_counter
	  (cm, thread_index, lb_index0, 1,
	  vlib_buffer_length_in_chain (vm, b[0]));
    vlib_increment_combined_counter
	  (cm, thread_index, lb_index1, 1,
	  vlib_buffer_length_in_chain (vm, b[1]));

    b += 2;
    next += 2;
    n_left -= 2;
  }
#endif
  while (n_left > 0)
  {
    ip4_header_t *ip0;
    const load_balance_t *lb0;
    ip4_fib_mtrie_t *mtrie0;
    ip4_fib_mtrie_leaf_t leaf0;
    ip4_address_t *dst_addr0;
    u32 lbi0;
    flow_hash_config_t flow_hash_config0;
    const dpo_id_t *dpo0;
    u32 hash_c0;

    ip0 = vlib_buffer_get_current (b[0]);
    dst_addr0 = &ip0->dst_address;
    ip_lookup_set_buffer_fib_index (im->fib_index_by_sw_if_index, b[0]);

    mtrie0 = &ip4_fib_get (vnet_buffer (b[0])->ip.fib_index)->mtrie;
    leaf0 = ip4_fib_mtrie_lookup_step_one (mtrie0, dst_addr0);
    leaf0 = ip4_fib_mtrie_lookup_step (mtrie0, leaf0, dst_addr0, 2);
    leaf0 = ip4_fib_mtrie_lookup_step (mtrie0, leaf0, dst_addr0, 3);
    lbi0 = ip4_fib_mtrie_leaf_get_adj_index (leaf0);

    ASSERT (lbi0);
    lb0 = load_balance_get (lbi0);

    ASSERT (lb0->lb_n_buckets > 0);
    ASSERT (is_pow2 (lb0->lb_n_buckets));

    /* Use flow hash to compute multipath adjacency. */
    hash_c0 = vnet_buffer (b[0])->ip.flow_hash = 0;
    if (PREDICT_FALSE (lb0->lb_n_buckets > 1))
	{
	  flow_hash_config0 = lb0->lb_hash_config;

	  hash_c0 = vnet_buffer (b[0])->ip.flow_hash =
	    ip4_compute_flow_hash (ip0, flow_hash_config0);
	  dpo0 =
	    load_balance_get_fwd_bucket (lb0,
					 (hash_c0 &
					  (lb0->lb_n_buckets_minus_1)));
	}
    else
	{
	  dpo0 = load_balance_get_bucket_i (lb0, 0);
	}

    next[0] = dpo0->dpoi_next_node;
    vnet_buffer (b[0])->ip.adj_index[VLIB_TX] = dpo0->dpoi_index;

    vlib_increment_combined_counter (cm, thread_index, lbi0, 1,
				       vlib_buffer_length_in_chain (vm,
								    b[0]));

    b += 1;
    next += 1;
    n_left -= 1;
  }

  /* 把frame传到下个node处理 */
  vlib_buffer_enqueue_to_next (vm, node, from, nexts, frame->n_vectors);

  if (node->flags & VLIB_NODE_FLAG_TRACE)
    ip4_forward_next_trace (vm, node, frame, VLIB_TX);

  return frame->n_vectors;
}
```

### ip4-rewrite

```
VLIB_REGISTER_NODE (ip4_rewrite_node) = {
  .name = "ip4-rewrite",
  .vector_size = sizeof (u32),

  .format_trace = format_ip4_rewrite_trace,

  .n_next_nodes = IP4_REWRITE_N_NEXT,
  .next_nodes = {
    [IP4_REWRITE_NEXT_DROP] = "ip4-drop",
    [IP4_REWRITE_NEXT_ICMP_ERROR] = "ip4-icmp-error",
    [IP4_REWRITE_NEXT_FRAGMENT] = "ip4-frag",
  },
};
```

```
/** @brief IPv4 rewrite node.
    @node ip4-rewrite

    This is the IPv4 transit-rewrite node: decrement TTL, fix the ipv4
    header checksum, fetch the ip adjacency, check the outbound mtu,
    apply the adjacency rewrite, and send pkts to the adjacency
    rewrite header's rewrite_next_index.

    @param vm vlib_main_t corresponding to the current thread
    @param node vlib_node_runtime_t
    @param frame vlib_frame_t whose contents should be dispatched

    @par Graph mechanics: buffer metadata, next index usage

    @em Uses:
    - <code>vnet_buffer(b)->ip.adj_index[VLIB_TX]</code>
        - the rewrite adjacency index
    - <code>adj->lookup_next_index</code>
        - Must be IP_LOOKUP_NEXT_REWRITE or IP_LOOKUP_NEXT_ARP, otherwise
          the packet will be dropped.
    - <code>adj->rewrite_header</code>
        - Rewrite string length, rewrite string, next_index

    @em Sets:
    - <code>b->current_data, b->current_length</code>
        - Updated net of applying the rewrite string

    <em>Next Indices:</em>
    - <code> adj->rewrite_header.next_index </code>
      or @c ip4-drop
*/

VLIB_NODE_FN (ip4_rewrite_node) (vlib_main_t * vm, vlib_node_runtime_t * node,
				 vlib_frame_t * frame)
{
  if (adj_are_counters_enabled ())
    return ip4_rewrite_inline (vm, node, frame, 1, 0, 0);
  else
    return ip4_rewrite_inline (vm, node, frame, 0, 0, 0);
}
```

```
always_inline uword
ip4_rewrite_inline (vlib_main_t * vm,
		    vlib_node_runtime_t * node,
		    vlib_frame_t * frame,
		    int do_counters, int is_midchain, int is_mcast)
{
  return ip4_rewrite_inline_with_gso (vm, node, frame, do_counters,
				      is_midchain, is_mcast);
}
```

```
always_inline uword
ip4_rewrite_inline_with_gso (vlib_main_t * vm,
			     vlib_node_runtime_t * node,
			     vlib_frame_t * frame,
			     int do_counters, int is_midchain, int is_mcast)
{
  ip_lookup_main_t *lm = &ip4_main.lookup_main;
  u32 *from = vlib_frame_vector_args (frame);
  vlib_buffer_t *bufs[VLIB_FRAME_SIZE], **b;
  u16 nexts[VLIB_FRAME_SIZE], *next;
  u32 n_left_from;
  vlib_node_runtime_t *error_node =
    vlib_node_get_runtime (vm, ip4_input_node.index);

  n_left_from = frame->n_vectors;
  u32 thread_index = vm->thread_index;

  vlib_get_buffers (vm, from, bufs, n_left_from);
  clib_memset_u16 (nexts, IP4_REWRITE_NEXT_DROP, n_left_from);

#if (CLIB_N_PREFETCHES >= 8)
  if (n_left_from >= 6)
  {
    int i;
    for (i = 2; i < 6; i++)
	  vlib_prefetch_buffer_header (bufs[i], LOAD);
  }

  next = nexts;
  b = bufs;
  while (n_left_from >= 8)
  {
    const ip_adjacency_t *adj0, *adj1;
    ip4_header_t *ip0, *ip1;
    u32 rw_len0, error0, adj_index0;
    u32 rw_len1, error1, adj_index1;
    u32 tx_sw_if_index0, tx_sw_if_index1;
    u8 *p;

    vlib_prefetch_buffer_header (b[6], LOAD);
    vlib_prefetch_buffer_header (b[7], LOAD);

    // 数据包在tx接口的adj索引
    adj_index0 = vnet_buffer (b[0])->ip.adj_index[VLIB_TX];
    adj_index1 = vnet_buffer (b[1])->ip.adj_index[VLIB_TX];

    /*
     * pre-fetch the per-adjacency counters
     */
    if (do_counters)
	{
	  vlib_prefetch_combined_counter (&adjacency_counters,
				  thread_index, adj_index0);
	  vlib_prefetch_combined_counter (&adjacency_counters,
				  thread_index, adj_index1);
	}

    ip0 = vlib_buffer_get_current (b[0]);
    ip1 = vlib_buffer_get_current (b[1]);

    error0 = error1 = IP4_ERROR_NONE;

    // 减小数据包ttl值，并更新校验和
    ip4_ttl_and_checksum_check (b[0], ip0, next + 0, &error0);
    ip4_ttl_and_checksum_check (b[1], ip1, next + 1, &error1);

    /* Rewrite packet header and updates lengths. */
    // 根据adj索引获取adj
    adj0 = adj_get (adj_index0);
    adj1 = adj_get (adj_index1);

    /* Worth pipelining. No guarantee that adj0,1 are hot... */
    // 从adj获取rewrite数据长度
    rw_len0 = adj0[0].rewrite_header.data_bytes;
    rw_len1 = adj1[0].rewrite_header.data_bytes;
    // 将rewrite数据长度保存到数据包中
    vnet_buffer (b[0])->ip.save_rewrite_length = rw_len0;
    vnet_buffer (b[1])->ip.save_rewrite_length = rw_len1;

    // PREFETCH后两个数据包
    p = vlib_buffer_get_current (b[2]);
    CLIB_PREFETCH (p - CLIB_CACHE_LINE_BYTES, CLIB_CACHE_LINE_BYTES, STORE);
    CLIB_PREFETCH (p, CLIB_CACHE_LINE_BYTES, LOAD);

    p = vlib_buffer_get_current (b[3]);
    CLIB_PREFETCH (p - CLIB_CACHE_LINE_BYTES, CLIB_CACHE_LINE_BYTES, STORE);
    CLIB_PREFETCH (p, CLIB_CACHE_LINE_BYTES, LOAD);

    /* Check MTU of outgoing interface. */
    // 获取ip数据包长度
    u16 ip0_len = clib_net_to_host_u16 (ip0->length);
    u16 ip1_len = clib_net_to_host_u16 (ip1->length);
    // 接口开启了gso，则修正ip数据包长度
    if (b[0]->flags & VNET_BUFFER_F_GSO)
	  ip0_len = gso_mtu_sz (b[0]);
    if (b[1]->flags & VNET_BUFFER_F_GSO)
	  ip1_len = gso_mtu_sz (b[1]);
    
    // 校验tx接口的mtu值，如果ip数据包长度大于mtu则需要进行分段
    // Max packet size layer 3 (MTU) for output interface. Used for MTU check after packet rewrite.
    ip4_mtu_check (b[0], ip0_len,
		     adj0[0].rewrite_header.max_l3_packet_bytes,
		     ip0->flags_and_fragment_offset &
		     clib_host_to_net_u16 (IP4_HEADER_FLAG_DONT_FRAGMENT),
		     next + 0, is_midchain, &error0);
    ip4_mtu_check (b[1], ip1_len,
		     adj1[0].rewrite_header.max_l3_packet_bytes,
		     ip1->flags_and_fragment_offset &
		     clib_host_to_net_u16 (IP4_HEADER_FLAG_DONT_FRAGMENT),
		     next + 1, is_midchain, &error1);

    // 多播，校验数据包rx接口不能等于adj指明的tx接口
    if (is_mcast)
	{
	  error0 = ((adj0[0].rewrite_header.sw_if_index ==
		     vnet_buffer (b[0])->sw_if_index[VLIB_RX]) ?
		    IP4_ERROR_SAME_INTERFACE : error0);
	  error1 = ((adj1[0].rewrite_header.sw_if_index ==
		     vnet_buffer (b[1])->sw_if_index[VLIB_RX]) ?
		    IP4_ERROR_SAME_INTERFACE : error1);
	}

    /* Don't adjust the buffer for ttl issue; icmp-error node wants
     * to see the IP header */
    // 无错误
    if (PREDICT_TRUE (error0 == IP4_ERROR_NONE))
	{
      // 获取rewrite后的next节点
	  u32 next_index = adj0[0].rewrite_header.next_index;
	  vlib_buffer_advance (b[0], -(word) rw_len0);

      // 由adj获取tx接口索引，并赋给数据包
	  tx_sw_if_index0 = adj0[0].rewrite_header.sw_if_index;
	  vnet_buffer (b[0])->sw_if_index[VLIB_TX] = tx_sw_if_index0;

      // This adjacency/interface has output features configured
      // 如果tx接口上启用了feature arc，则开始执行featrue arc
	  if (PREDICT_FALSE (adj0[0].rewrite_header.flags & VNET_REWRITE_HAS_FEATURES))
	    vnet_feature_arc_start (lm->output_feature_arc_index, tx_sw_if_index0, &next_index, b[0]);
	  next[0] = next_index;
	  if (is_midchain)
        /* this acts on the packet that is about to be encapped */
	    calc_checksums (vm, b[0]);
	}
    // 有错误发生
    else
	{
	  b[0]->error = error_node->errors[error0];
      // 大于mtu，则增加ttl值更新校验和，why???
	  if (error0 == IP4_ERROR_MTU_EXCEEDED)
	    ip4_ttl_inc (b[0], ip0);
	}
    if (PREDICT_TRUE (error1 == IP4_ERROR_NONE))
	{
	  u32 next_index = adj1[0].rewrite_header.next_index;
	  vlib_buffer_advance (b[1], -(word) rw_len1);

	  tx_sw_if_index1 = adj1[0].rewrite_header.sw_if_index;
	  vnet_buffer (b[1])->sw_if_index[VLIB_TX] = tx_sw_if_index1;

	  if (PREDICT_FALSE
	      (adj1[0].rewrite_header.flags & VNET_REWRITE_HAS_FEATURES))
	    vnet_feature_arc_start (lm->output_feature_arc_index,
				    tx_sw_if_index1, &next_index, b[1]);
	  next[1] = next_index;
	  if (is_midchain)
	    calc_checksums (vm, b[1]);
	}
    else
	{
	  b[1]->error = error_node->errors[error1];
	  if (error1 == IP4_ERROR_MTU_EXCEEDED)
	    ip4_ttl_inc (b[1], ip1);
	}

    /* Guess we are only writing on simple Ethernet header. */
    // rewrite以太网头
    vnet_rewrite_two_headers (adj0[0], adj1[0],
				ip0, ip1, sizeof (ethernet_header_t));

    if (do_counters)
	{
	  if (error0 == IP4_ERROR_NONE)
	    vlib_increment_combined_counter
	      (&adjacency_counters,
	       thread_index,
	       adj_index0, 1,
	       vlib_buffer_length_in_chain (vm, b[0]) + rw_len0);

	  if (error1 == IP4_ERROR_NONE)
	    vlib_increment_combined_counter
	      (&adjacency_counters,
	       thread_index,
	       adj_index1, 1,
	       vlib_buffer_length_in_chain (vm, b[1]) + rw_len1);
	}

    if (is_midchain)
	{
	  if (error0 == IP4_ERROR_NONE && adj0->sub_type.midchain.fixup_func)
	    adj0->sub_type.midchain.fixup_func
	      (vm, adj0, b[0], adj0->sub_type.midchain.fixup_data);
	  if (error1 == IP4_ERROR_NONE && adj1->sub_type.midchain.fixup_func)
	    adj1->sub_type.midchain.fixup_func
	      (vm, adj1, b[1], adj1->sub_type.midchain.fixup_data);
	}

    if (is_mcast)
	{
	  /* copy bytes from the IP address into the MAC rewrite */
	  if (error0 == IP4_ERROR_NONE)
	    vnet_ip_mcast_fixup_header (IP4_MCAST_ADDR_MASK,
					adj0->rewrite_header.dst_mcast_offset,
					&ip0->dst_address.as_u32, (u8 *) ip0);
	  if (error1 == IP4_ERROR_NONE)
	    vnet_ip_mcast_fixup_header (IP4_MCAST_ADDR_MASK,
					adj1->rewrite_header.dst_mcast_offset,
					&ip1->dst_address.as_u32, (u8 *) ip1);
	}

    next += 2;
    b += 2;
    n_left_from -= 2;
  }
#elif (CLIB_N_PREFETCHES >= 4)
  next = nexts;
  b = bufs;
  while (n_left_from >= 1)
  {
    ip_adjacency_t *adj0;
    ip4_header_t *ip0;
    u32 rw_len0, error0, adj_index0;
    u32 tx_sw_if_index0;
    u8 *p;

    /* Prefetch next iteration */
    if (PREDICT_TRUE (n_left_from >= 4))
	{
	  ip_adjacency_t *adj2;
	  u32 adj_index2;

	  vlib_prefetch_buffer_header (b[3], LOAD);
	  vlib_prefetch_buffer_data (b[2], LOAD);

	  /* Prefetch adj->rewrite_header */
	  adj_index2 = vnet_buffer (b[2])->ip.adj_index[VLIB_TX];
	  adj2 = adj_get (adj_index2);
	  p = (u8 *) adj2;
	  CLIB_PREFETCH (p + CLIB_CACHE_LINE_BYTES, CLIB_CACHE_LINE_BYTES,
			 LOAD);
	}

    adj_index0 = vnet_buffer (b[0])->ip.adj_index[VLIB_TX];

    /*
     * Prefetch the per-adjacency counters
     */
    if (do_counters)
	{
	  vlib_prefetch_combined_counter (&adjacency_counters,
					  thread_index, adj_index0);
	}

    ip0 = vlib_buffer_get_current (b[0]);

    error0 = IP4_ERROR_NONE;

    ip4_ttl_and_checksum_check (b[0], ip0, next + 0, &error0);

    /* Rewrite packet header and updates lengths. */
    adj0 = adj_get (adj_index0);

    /* Rewrite header was prefetched. */
    rw_len0 = adj0[0].rewrite_header.data_bytes;
    vnet_buffer (b[0])->ip.save_rewrite_length = rw_len0;

    /* Check MTU of outgoing interface. */
    u16 ip0_len = clib_net_to_host_u16 (ip0->length);

    if (b[0]->flags & VNET_BUFFER_F_GSO)
	  ip0_len = gso_mtu_sz (b[0]);

    ip4_mtu_check (b[0], ip0_len,
		     adj0[0].rewrite_header.max_l3_packet_bytes,
		     ip0->flags_and_fragment_offset &
		     clib_host_to_net_u16 (IP4_HEADER_FLAG_DONT_FRAGMENT),
		     next + 0, is_midchain, &error0);

    if (is_mcast)
	{
	  error0 = ((adj0[0].rewrite_header.sw_if_index ==
		     vnet_buffer (b[0])->sw_if_index[VLIB_RX]) ?
		    IP4_ERROR_SAME_INTERFACE : error0);
	}

    /* Don't adjust the buffer for ttl issue; icmp-error node wants
     * to see the IP header */
    if (PREDICT_TRUE (error0 == IP4_ERROR_NONE))
	{
	  u32 next_index = adj0[0].rewrite_header.next_index;
	  vlib_buffer_advance (b[0], -(word) rw_len0);
	  tx_sw_if_index0 = adj0[0].rewrite_header.sw_if_index;
	  vnet_buffer (b[0])->sw_if_index[VLIB_TX] = tx_sw_if_index0;

	  if (PREDICT_FALSE
	      (adj0[0].rewrite_header.flags & VNET_REWRITE_HAS_FEATURES))
	    vnet_feature_arc_start (lm->output_feature_arc_index,
				    tx_sw_if_index0, &next_index, b[0]);
	  next[0] = next_index;

	  if (is_midchain)
	    calc_checksums (vm, b[0]);

	  /* Guess we are only writing on simple Ethernet header. */
	  vnet_rewrite_one_header (adj0[0], ip0, sizeof (ethernet_header_t));

	  /*
	   * Bump the per-adjacency counters
	   */
	  if (do_counters)
	    vlib_increment_combined_counter
	      (&adjacency_counters,
	       thread_index,
	       adj_index0, 1, vlib_buffer_length_in_chain (vm,
							   b[0]) + rw_len0);

	  if (is_midchain && adj0->sub_type.midchain.fixup_func)
	    adj0->sub_type.midchain.fixup_func
	      (vm, adj0, b[0], adj0->sub_type.midchain.fixup_data);

	  if (is_mcast)
	    /* copy bytes from the IP address into the MAC rewrite */
	    vnet_ip_mcast_fixup_header (IP4_MCAST_ADDR_MASK,
					adj0->rewrite_header.dst_mcast_offset,
					&ip0->dst_address.as_u32, (u8 *) ip0);
	}
    else
	{
	  b[0]->error = error_node->errors[error0];
	  if (error0 == IP4_ERROR_MTU_EXCEEDED)
	    ip4_ttl_inc (b[0], ip0);
	}

    next += 1;
    b += 1;
    n_left_from -= 1;
  }
#endif

  while (n_left_from > 0)
  {
    ip_adjacency_t *adj0;
    ip4_header_t *ip0;
    u32 rw_len0, adj_index0, error0;
    u32 tx_sw_if_index0;

    adj_index0 = vnet_buffer (b[0])->ip.adj_index[VLIB_TX];

    adj0 = adj_get (adj_index0);

    if (do_counters)
      vlib_prefetch_combined_counter (&adjacency_counters,
				thread_index, adj_index0);

    ip0 = vlib_buffer_get_current (b[0]);

    error0 = IP4_ERROR_NONE;

    ip4_ttl_and_checksum_check (b[0], ip0, next + 0, &error0);


    /* Update packet buffer attributes/set output interface. */
    rw_len0 = adj0[0].rewrite_header.data_bytes;
    vnet_buffer (b[0])->ip.save_rewrite_length = rw_len0;

    /* Check MTU of outgoing interface. */
    u16 ip0_len = clib_net_to_host_u16 (ip0->length);
    if (b[0]->flags & VNET_BUFFER_F_GSO)
      ip0_len = gso_mtu_sz (b[0]);

    ip4_mtu_check (b[0], ip0_len,
	     adj0[0].rewrite_header.max_l3_packet_bytes,
	     ip0->flags_and_fragment_offset &
	     clib_host_to_net_u16 (IP4_HEADER_FLAG_DONT_FRAGMENT),
	     next + 0, is_midchain, &error0);

    if (is_mcast)
	{
	  error0 = ((adj0[0].rewrite_header.sw_if_index ==
		     vnet_buffer (b[0])->sw_if_index[VLIB_RX]) ?
		    IP4_ERROR_SAME_INTERFACE : error0);
	}

    /* Don't adjust the buffer for ttl issue; icmp-error node wants
     * to see the IP header */
    if (PREDICT_TRUE (error0 == IP4_ERROR_NONE))
	{
	  u32 next_index = adj0[0].rewrite_header.next_index;
	  vlib_buffer_advance (b[0], -(word) rw_len0);
	  tx_sw_if_index0 = adj0[0].rewrite_header.sw_if_index;
	  vnet_buffer (b[0])->sw_if_index[VLIB_TX] = tx_sw_if_index0;

	  if (PREDICT_FALSE
	      (adj0[0].rewrite_header.flags & VNET_REWRITE_HAS_FEATURES))
	    vnet_feature_arc_start (lm->output_feature_arc_index,
				    tx_sw_if_index0, &next_index, b[0]);
	  next[0] = next_index;

	  if (is_midchain)
	    /* this acts on the packet that is about to be encapped */
	    calc_checksums (vm, b[0]);

	  /* Guess we are only writing on simple Ethernet header. */
	  vnet_rewrite_one_header (adj0[0], ip0, sizeof (ethernet_header_t));

	  if (do_counters)
	    vlib_increment_combined_counter
	      (&adjacency_counters,
	       thread_index, adj_index0, 1,
	       vlib_buffer_length_in_chain (vm, b[0]) + rw_len0);

	  if (is_midchain && adj0->sub_type.midchain.fixup_func)
	    adj0->sub_type.midchain.fixup_func
	      (vm, adj0, b[0], adj0->sub_type.midchain.fixup_data);

	  if (is_mcast)
	    /* copy bytes from the IP address into the MAC rewrite */
	    vnet_ip_mcast_fixup_header (IP4_MCAST_ADDR_MASK,
					adj0->rewrite_header.dst_mcast_offset,
					&ip0->dst_address.as_u32, (u8 *) ip0);
	}
    else
	{
	  b[0]->error = error_node->errors[error0];
	  /* undo the TTL decrement - we'll be back to do it again */
	  if (error0 == IP4_ERROR_MTU_EXCEEDED)
	    ip4_ttl_inc (b[0], ip0);
	}

    next += 1;
    b += 1;
    n_left_from -= 1;
  }

  /* Need to do trace after rewrites to pick up new packet data. */
  if (node->flags & VLIB_NODE_FLAG_TRACE)
    ip4_forward_next_trace (vm, node, frame, VLIB_TX);

  vlib_buffer_enqueue_to_next (vm, node, from, nexts, frame->n_vectors);
  return frame->n_vectors;
}
```

### ip4-local

### ip4-load-balance