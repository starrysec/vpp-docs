## ip

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