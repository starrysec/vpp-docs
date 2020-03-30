## ethernet

### 处理

#### vnet/ethernet/node.c

```
VLIB_REGISTER_NODE (ethernet_input_node) = {
  .name = "ethernet-input",
  /* Takes a vector of packets. */
  .vector_size = sizeof (u32),
  .scalar_size = sizeof (ethernet_input_frame_t),
  .n_errors = ETHERNET_N_ERROR,
  .error_strings = ethernet_error_strings,
  .n_next_nodes = ETHERNET_INPUT_N_NEXT,
  .next_nodes = {
#define _(s,n) [ETHERNET_INPUT_NEXT_##s] = n,
    foreach_ethernet_input_next
#undef _
  },
  .format_buffer = format_ethernet_header_with_length,
  .format_trace = format_ethernet_input_trace,
  .unformat_buffer = unformat_ethernet_header,
};
```

```
VLIB_NODE_FN (ethernet_input_node) (vlib_main_t * vm,
				    vlib_node_runtime_t * node,
				    vlib_frame_t * frame)
{
  vnet_main_t *vnm = vnet_get_main ();
  u32 *from = vlib_frame_vector_args (frame);
  u32 n_packets = frame->n_vectors;

  ethernet_input_trace (vm, node, frame);

  /* 检查frame标记：所有数据包都是同一个接口，一般是没有物理接口的虚拟设备产生的数据包 */
  if (frame->flags & ETH_INPUT_FRAME_F_SINGLE_SW_IF_IDX)
    {
      ethernet_input_frame_t *ef = vlib_frame_scalar_args (frame);
      /* 校验和正确标记，设置了就不用在检查了(软件产生的数据包校验和都正确的) */
      int ip4_cksum_ok = (frame->flags & ETH_INPUT_FRAME_F_IP4_CKSUM_OK) != 0;
      vnet_hw_interface_t *hi = vnet_get_hw_interface (vnm, ef->hw_if_index);
      eth_input_single_int (vm, node, hi, from, n_packets, ip4_cksum_ok);
    }
  /* 正常数据包处理流程 */
  else
    ethernet_input_inline (vm, node, from, n_packets,
			   ETHERNET_INPUT_VARIANT_ETHERNET);
  return n_packets;
}
```

```
static_always_inline void
eth_input_single_int (vlib_main_t * vm, vlib_node_runtime_t * node,
		      vnet_hw_interface_t * hi, u32 * from, u32 n_pkts,
		      int ip4_cksum_ok)
{
  ethernet_main_t *em = &ethernet_main;
  ethernet_interface_t *ei;
  ei = pool_elt_at_index (em->interfaces, hi->hw_instance);
  /* 获取主接口 */
  main_intf_t *intf0 = vec_elt_at_index (em->main_intfs, hi->hw_if_index);
  /* 获取untagged（没有vlan tag）子接口配置 */
  subint_config_t *subint0 = &intf0->untagged_subint;

  /* 接口是L2模式还是L3模式 */
  int main_is_l3 = (subint0->flags & SUBINT_CONFIG_L2) == 0;
  /* 接口是否是混杂模式 */
  int promisc = (ei->flags & ETHERNET_INTERFACE_FLAG_ACCEPT_ALL) != 0;

  /* L3模式 */
  if (main_is_l3)
    {
      /* main interface is L3, we dont expect tagged packets and interface
         is not in promisc node, so we dont't need to check DMAC */
      int is_l3 = 1;

      /* 不是混杂模式，不需要检查dest mac */
      if (promisc == 0)
	    eth_input_process_frame (vm, node, hi, from, n_pkts, is_l3,
				 ip4_cksum_ok, 0);
      /* 混杂模式，需要检查dest mac */
      else
	    /* subinterfaces and promisc mode so DMAC check is needed */
	    eth_input_process_frame (vm, node, hi, from, n_pkts, is_l3,
				 ip4_cksum_ok, 1);
      return;
    }
  /* L2模式 */
  else
    {
      /* untagged packets are treated as L2 */
      int is_l3 = 0;
      /* 按L2处理数据包 */
      eth_input_process_frame (vm, node, hi, from, n_pkts, is_l3,
			       ip4_cksum_ok, 1);
      return;
    }
}
```

```
static_always_inline void
eth_input_process_frame (vlib_main_t * vm, vlib_node_runtime_t * node,
			 vnet_hw_interface_t * hi,
			 u32 * buffer_indices, u32 n_packets, int main_is_l3,
			 int ip4_cksum_ok, int dmac_check)
{
  ethernet_main_t *em = &ethernet_main;
  u16 nexts[VLIB_FRAME_SIZE], *next;
  u16 etypes[VLIB_FRAME_SIZE], *etype = etypes;
  u64 dmacs[VLIB_FRAME_SIZE], *dmac = dmacs;
  u8 dmacs_bad[VLIB_FRAME_SIZE];
  u64 tags[VLIB_FRAME_SIZE], *tag = tags;
  u16 slowpath_indices[VLIB_FRAME_SIZE];
  u16 n_slowpath, i;
  u16 next_ip4, next_ip6, next_mpls, next_l2;
  u16 et_ip4 = clib_host_to_net_u16 (ETHERNET_TYPE_IP4);
  u16 et_ip6 = clib_host_to_net_u16 (ETHERNET_TYPE_IP6);
  u16 et_mpls = clib_host_to_net_u16 (ETHERNET_TYPE_MPLS);
  u16 et_vlan = clib_host_to_net_u16 (ETHERNET_TYPE_VLAN);
  u16 et_dot1ad = clib_host_to_net_u16 (ETHERNET_TYPE_DOT1AD);
  i32 n_left = n_packets;
  vlib_buffer_t *b[20];
  u32 *from;
  /* 通过硬件接口索引，获取以太网接口实例 */
  ethernet_interface_t *ei = ethernet_get_interface (em, hi->hw_if_index);

  from = buffer_indices;

  while (n_left >= 20)
  {
      vlib_buffer_t **ph = b + 16, **pd = b + 8;
      /* 索引转地址，from开始1-4数据包，存入b[0]-b[3] */
      vlib_get_buffers (vm, from, b, 4);
      /* 索引转地址，from开始9-12数据包，存入pd[0]-pd[3]，即b[8]-b[11] */
      vlib_get_buffers (vm, from + 8, pd, 4);
      /* 索引转地址，from开始17-20数据包，存入ph[0]-ph[3]，即b[16]-b[19] */
      vlib_get_buffers (vm, from + 16, ph, 4);

      /* 预取ph[0]的64字节头 */
      vlib_prefetch_buffer_header (ph[0], LOAD);
      /* 预取ph[0]的数据 */
      vlib_prefetch_buffer_data (pd[0], LOAD);
      /* 获取以太网类型和tag(如vlan)，如果要校验dmac则也获取dest mac地址 */
      eth_input_get_etype_and_tags (b, etype, tag, dmac, 0, dmac_check);

      vlib_prefetch_buffer_header (ph[1], LOAD);
      vlib_prefetch_buffer_data (pd[1], LOAD);
      eth_input_get_etype_and_tags (b, etype, tag, dmac, 1, dmac_check);

      vlib_prefetch_buffer_header (ph[2], LOAD);
      vlib_prefetch_buffer_data (pd[2], LOAD);
      eth_input_get_etype_and_tags (b, etype, tag, dmac, 2, dmac_check);

      vlib_prefetch_buffer_header (ph[3], LOAD);
      vlib_prefetch_buffer_data (pd[3], LOAD);
      eth_input_get_etype_and_tags (b, etype, tag, dmac, 3, dmac_check);

      eth_input_adv_and_flags_x4 (b, main_is_l3);

      /* next */
      n_left -= 4;
      etype += 4;
      tag += 4;
      dmac += 4;
      from += 4;
  }
  while (n_left >= 4) 
  {
      /* 索引转地址，from开始1-4数据包，存入b[0]-b[3] */
      vlib_get_buffers (vm, from, b, 4);
      eth_input_get_etype_and_tags (b, etype, tag, dmac, 0, dmac_check);
      eth_input_get_etype_and_tags (b, etype, tag, dmac, 1, dmac_check);
      eth_input_get_etype_and_tags (b, etype, tag, dmac, 2, dmac_check);
      eth_input_get_etype_and_tags (b, etype, tag, dmac, 3, dmac_check);
      eth_input_adv_and_flags_x4 (b, main_is_l3);

      /* next */
      n_left -= 4;
      etype += 4;
      tag += 4;
      dmac += 4;
      from += 4;
  }
  while (n_left)
  {
      vlib_get_buffers (vm, from, b, 1);
      eth_input_get_etype_and_tags (b, etype, tag, dmac, 0, dmac_check);
      eth_input_adv_and_flags_x1 (b, main_is_l3);

      /* next */
      n_left -= 1;
      etype += 1;
      tag += 1;
      dmac += 4;
      from += 1;
  }

  /* 检查dest mac */
  if (dmac_check)
  {
      /* 以太网接口有secondary mac地址 */
      if (ei && vec_len (ei->secondary_addrs))
        /* 检查数据包dest mac和硬件接口mac地址是否匹配 */
	    eth_input_process_frame_dmac_check (hi, dmacs, dmacs_bad, n_packets,
					    ei, 1 /* have_sec_dmac */ );
      /* 以太网接口没有secondary mac地址 */
      else
        /* 检查数据包dest mac和硬件接口mac地址是否匹配(包含secondary mac检查) */
	    eth_input_process_frame_dmac_check (hi, dmacs, dmacs_bad, n_packets,
					    ei, 0 /* have_sec_dmac */ );
  }
  
  /* 根据以太网类型，获取下个L3 input节点索引 */
  next_ip4 = em->l3_next.input_next_ip4;
  next_ip6 = em->l3_next.input_next_ip6;
  next_mpls = em->l3_next.input_next_mpls;
  /* 根据以太网类型，获取下个L2 input节点索引 */
  next_l2 = em->l2_next;

  /* 已设置ipv4校验和正确，则不需再校验 */
  if (next_ip4 == ETHERNET_INPUT_NEXT_IP4_INPUT && ip4_cksum_ok)
    next_ip4 = ETHERNET_INPUT_NEXT_IP4_INPUT_NCS;

#ifdef CLIB_HAVE_VEC256
  u16x16 et16_ip4 = u16x16_splat (et_ip4);
  u16x16 et16_ip6 = u16x16_splat (et_ip6);
  u16x16 et16_mpls = u16x16_splat (et_mpls);
  u16x16 et16_vlan = u16x16_splat (et_vlan);
  u16x16 et16_dot1ad = u16x16_splat (et_dot1ad);
  u16x16 next16_ip4 = u16x16_splat (next_ip4);
  u16x16 next16_ip6 = u16x16_splat (next_ip6);
  u16x16 next16_mpls = u16x16_splat (next_mpls);
  u16x16 next16_l2 = u16x16_splat (next_l2);
  u16x16 zero = { 0 };
  u16x16 stairs = { 0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15 };
#endif

  /* etype是etypes数组的首地址 */
  etype = etypes;
  n_left = n_packets;
  next = nexts;
  n_slowpath = 0;
  i = 0;

  /* fastpath - in l3 mode hadles ip4, ip6 and mpls packets, other packets
     are considered as slowpath, in l2 mode all untagged packets are
     considered as fastpath */
  while (n_left > 0)
    {
#ifdef CLIB_HAVE_VEC256
      if (n_left >= 16)
	{
	  u16x16 r = zero;
	  u16x16 e16 = u16x16_load_unaligned (etype);
	  if (main_is_l3)
	    {
	      r += (e16 == et16_ip4) & next16_ip4;
	      r += (e16 == et16_ip6) & next16_ip6;
	      r += (e16 == et16_mpls) & next16_mpls;
	    }
	  else
	    r = ((e16 != et16_vlan) & (e16 != et16_dot1ad)) & next16_l2;
	  u16x16_store_unaligned (r, next);

	  if (!u16x16_is_all_zero (r == zero))
	    {
	      if (u16x16_is_all_zero (r))
		{
		  u16x16_store_unaligned (u16x16_splat (i) + stairs,
					  slowpath_indices + n_slowpath);
		  n_slowpath += 16;
		}
	      else
		{
		  for (int j = 0; j < 16; j++)
		    if (next[j] == 0)
		      slowpath_indices[n_slowpath++] = i + j;
		}
	    }

	  etype += 16;
	  next += 16;
	  n_left -= 16;
	  i += 16;
	  continue;
	}
#endif
      /* L3模式，以太网类型ipv4，则fastpath */
      if (main_is_l3 && etype[0] == et_ip4)
	    next[0] = next_ip4;
      /* L3模式，以太网类型ipv6，则fastpath */
      else if (main_is_l3 && etype[0] == et_ip6)
	    next[0] = next_ip6;
      /* L3模式，以太网类型mpls，则fastpath */
      else if (main_is_l3 && etype[0] == et_mpls)
	    next[0] = next_mpls;
      /* L2模式，untagged，则fastpath */
      else if (main_is_l3 == 0 && etype[0] != et_vlan && etype[0] != et_dot1ad)
	    next[0] = next_l2;
      /* 其他，slowpath */
      else
        {
          next[0] = 0;
          slowpath_indices[n_slowpath++] = i;
        }

      etype += 1;
      next += 1;
      n_left -= 1;
      i += 1;
    }

  /* 有需要slowpath处理的数据包 */
  if (n_slowpath)
    {
      vnet_main_t *vnm = vnet_get_main ();
      n_left = n_slowpath;
      u16 *si = slowpath_indices;
      u32 last_unknown_etype = ~0;
      u32 last_unknown_next = ~0;
      /* 初始化dot1q_lookup结构体 */
      eth_input_tag_lookup_t dot1ad_lookup, dot1q_lookup = {
	    .mask = -1LL,
	    .tag = tags[si[0]] ^ -1LL,
	    .sw_if_index = ~0
      };
      /* 初始化dot1ad_lookup结构体 */
      clib_memcpy_fast (&dot1ad_lookup, &dot1q_lookup, sizeof (dot1q_lookup));

      while (n_left)
	{
	  i = si[0];
      /* 读取以太网类型 */
	  u16 etype = etypes[i];

      /* 802.1q(vlan) */
	  if (etype == et_vlan)
	    {
	      vlib_buffer_t *b = vlib_get_buffer (vm, buffer_indices[i]);
          /* 检查vlan tag是否匹配subinterface接口 */
	      eth_input_tag_lookup (vm, vnm, node, hi, tags[i], nexts + i, b,
				    &dot1q_lookup, dmacs_bad[i], 0,
				    main_is_l3, dmac_check);

	    }
      /* 802.1ad(qinq) */
	  else if (etype == et_dot1ad)
	    {
	      vlib_buffer_t *b = vlib_get_buffer (vm, buffer_indices[i]);
          /* 检查qinq tag是否匹配subinterface接口 */
	      eth_input_tag_lookup (vm, vnm, node, hi, tags[i], nexts + i, b,
				    &dot1ad_lookup, dmacs_bad[i], 1,
				    main_is_l3, dmac_check);
	    }
      /* 其他无tag类型 */
	  else
	    {
	      /* untagged packet with not well known etyertype */
          /* 位置以太网类型 */
	      if (last_unknown_etype != etype)
		  {
		    last_unknown_etype = etype;
            /* 字节序转换 */
		    etype = clib_host_to_net_u16 (etype);
            /* next节点 */
		    last_unknown_next = eth_input_next_by_type (etype);
		  }

          /* 验证dest mac，L3模式，dmac校验失败 */
	      if (dmac_check && main_is_l3 && dmacs_bad[i])
		  {
            /* 置数据包错误信息，next节点为punt */
		    vlib_buffer_t *b = vlib_get_buffer (vm, buffer_indices[i]);
		    b->error = node->errors[ETHERNET_ERROR_L3_MAC_MISMATCH];
		    nexts[i] = ETHERNET_INPUT_NEXT_PUNT;
		  }
          /* 其他 */
	      else
		    nexts[i] = last_unknown_next;
	    }

	  /* next */
	  n_left--;
	  si++;
	}
      
      /* 更新packets，bytes等计数器 */
      eth_input_update_if_counters (vm, vnm, &dot1q_lookup);
      eth_input_update_if_counters (vm, vnm, &dot1ad_lookup);
    }

    /* 把frame传到下个node处理 */
    vlib_buffer_enqueue_to_next (vm, node, buffer_indices, nexts, n_packets);
}
```

```
static_always_inline void
ethernet_input_inline (vlib_main_t * vm,
		       vlib_node_runtime_t * node,
		       u32 * from, u32 n_packets,
		       ethernet_input_variant_t variant)
{
  vnet_main_t *vnm = vnet_get_main ();
  ethernet_main_t *em = &ethernet_main;
  vlib_node_runtime_t *error_node;
  u32 n_left_from, next_index, *to_next;
  u32 stats_sw_if_index, stats_n_packets, stats_n_bytes;
  u32 thread_index = vm->thread_index;
  u32 cached_sw_if_index = ~0;
  u32 cached_is_l2 = 0;		/* shut up gcc */
  vnet_hw_interface_t *hi = NULL;	/* used for main interface only */
  ethernet_interface_t *ei = NULL;
  vlib_buffer_t *bufs[VLIB_FRAME_SIZE];
  vlib_buffer_t **b = bufs;

  if (variant != ETHERNET_INPUT_VARIANT_ETHERNET)
    error_node = vlib_node_get_runtime (vm, ethernet_input_node.index);
  else
    error_node = node;

  n_left_from = n_packets;

  next_index = node->cached_next_index;
  stats_sw_if_index = node->runtime_data[0];
  stats_n_packets = stats_n_bytes = 0;
  vlib_get_buffers (vm, from, bufs, n_left_from);

  while (n_left_from > 0)
    {
      u32 n_left_to_next;

      vlib_get_next_frame (vm, node, next_index, to_next, n_left_to_next);

      while (n_left_from >= 4 && n_left_to_next >= 2)
	{
	  u32 bi0, bi1;
	  vlib_buffer_t *b0, *b1;
	  u8 next0, next1, error0, error1;
	  u16 type0, orig_type0, type1, orig_type1;
	  u16 outer_id0, inner_id0, outer_id1, inner_id1;
	  u32 match_flags0, match_flags1;
	  u32 old_sw_if_index0, new_sw_if_index0, len0, old_sw_if_index1,
	    new_sw_if_index1, len1;
	  vnet_hw_interface_t *hi0, *hi1;
	  main_intf_t *main_intf0, *main_intf1;
	  vlan_intf_t *vlan_intf0, *vlan_intf1;
	  qinq_intf_t *qinq_intf0, *qinq_intf1;
	  u32 is_l20, is_l21;
	  ethernet_header_t *e0, *e1;
	  u64 dmacs[2];
	  u8 dmacs_bad[2];

	  /* Prefetch next iteration. */
	  {
	    vlib_prefetch_buffer_header (b[2], STORE);
	    vlib_prefetch_buffer_header (b[3], STORE);

	    CLIB_PREFETCH (b[2]->data, sizeof (ethernet_header_t), LOAD);
	    CLIB_PREFETCH (b[3]->data, sizeof (ethernet_header_t), LOAD);
	  }

	  bi0 = from[0];
	  bi1 = from[1];
	  to_next[0] = bi0;
	  to_next[1] = bi1;
	  from += 2;
	  to_next += 2;
	  n_left_to_next -= 2;
	  n_left_from -= 2;

	  b0 = b[0];
	  b1 = b[1];
	  b += 2;

	  error0 = error1 = ETHERNET_ERROR_NONE;
	  e0 = vlib_buffer_get_current (b0);
	  type0 = clib_net_to_host_u16 (e0->type);
	  e1 = vlib_buffer_get_current (b1);
	  type1 = clib_net_to_host_u16 (e1->type);

	  /* Set the L2 header offset for all packets */
	  vnet_buffer (b0)->l2_hdr_offset = b0->current_data;
	  vnet_buffer (b1)->l2_hdr_offset = b1->current_data;
	  b0->flags |= VNET_BUFFER_F_L2_HDR_OFFSET_VALID;
	  b1->flags |= VNET_BUFFER_F_L2_HDR_OFFSET_VALID;

	  /* Speed-path for the untagged case */
	  if (PREDICT_TRUE (variant == ETHERNET_INPUT_VARIANT_ETHERNET
			    && !ethernet_frame_is_any_tagged_x2 (type0,
								 type1)))
	    {
	      main_intf_t *intf0;
	      subint_config_t *subint0;
	      u32 sw_if_index0, sw_if_index1;

	      sw_if_index0 = vnet_buffer (b0)->sw_if_index[VLIB_RX];
	      sw_if_index1 = vnet_buffer (b1)->sw_if_index[VLIB_RX];
	      is_l20 = cached_is_l2;

	      /* This is probably wholly unnecessary */
	      if (PREDICT_FALSE (sw_if_index0 != sw_if_index1))
		goto slowpath;

	      /* Now sw_if_index0 == sw_if_index1  */
	      if (PREDICT_FALSE (cached_sw_if_index != sw_if_index0))
		{
		  cached_sw_if_index = sw_if_index0;
		  hi = vnet_get_sup_hw_interface (vnm, sw_if_index0);
		  ei = ethernet_get_interface (em, hi->hw_if_index);
		  intf0 = vec_elt_at_index (em->main_intfs, hi->hw_if_index);
		  subint0 = &intf0->untagged_subint;
		  cached_is_l2 = is_l20 = subint0->flags & SUBINT_CONFIG_L2;
		}

	      if (PREDICT_TRUE (is_l20 != 0))
		{
		  vnet_buffer (b0)->l3_hdr_offset =
		    vnet_buffer (b0)->l2_hdr_offset +
		    sizeof (ethernet_header_t);
		  vnet_buffer (b1)->l3_hdr_offset =
		    vnet_buffer (b1)->l2_hdr_offset +
		    sizeof (ethernet_header_t);
		  b0->flags |= VNET_BUFFER_F_L3_HDR_OFFSET_VALID;
		  b1->flags |= VNET_BUFFER_F_L3_HDR_OFFSET_VALID;
		  next0 = em->l2_next;
		  vnet_buffer (b0)->l2.l2_len = sizeof (ethernet_header_t);
		  next1 = em->l2_next;
		  vnet_buffer (b1)->l2.l2_len = sizeof (ethernet_header_t);
		}
	      else
		{
		  dmacs[0] = *(u64 *) e0;
		  dmacs[1] = *(u64 *) e1;

		  if (ei && vec_len (ei->secondary_addrs))
		    ethernet_input_inline_dmac_check (hi, dmacs,
						      dmacs_bad,
						      2 /* n_packets */ ,
						      ei,
						      1 /* have_sec_dmac */ );
		  else
		    ethernet_input_inline_dmac_check (hi, dmacs,
						      dmacs_bad,
						      2 /* n_packets */ ,
						      ei,
						      0 /* have_sec_dmac */ );

		  if (dmacs_bad[0])
		    error0 = ETHERNET_ERROR_L3_MAC_MISMATCH;
		  if (dmacs_bad[1])
		    error1 = ETHERNET_ERROR_L3_MAC_MISMATCH;

		  vlib_buffer_advance (b0, sizeof (ethernet_header_t));
		  determine_next_node (em, variant, 0, type0, b0,
				       &error0, &next0);
		  vlib_buffer_advance (b1, sizeof (ethernet_header_t));
		  determine_next_node (em, variant, 0, type1, b1,
				       &error1, &next1);
		}
	      goto ship_it01;
	    }

	  /* Slow-path for the tagged case */
	slowpath:
	  parse_header (variant,
			b0,
			&type0,
			&orig_type0, &outer_id0, &inner_id0, &match_flags0);

	  parse_header (variant,
			b1,
			&type1,
			&orig_type1, &outer_id1, &inner_id1, &match_flags1);

	  old_sw_if_index0 = vnet_buffer (b0)->sw_if_index[VLIB_RX];
	  old_sw_if_index1 = vnet_buffer (b1)->sw_if_index[VLIB_RX];

	  eth_vlan_table_lookups (em,
				  vnm,
				  old_sw_if_index0,
				  orig_type0,
				  outer_id0,
				  inner_id0,
				  &hi0,
				  &main_intf0, &vlan_intf0, &qinq_intf0);

	  eth_vlan_table_lookups (em,
				  vnm,
				  old_sw_if_index1,
				  orig_type1,
				  outer_id1,
				  inner_id1,
				  &hi1,
				  &main_intf1, &vlan_intf1, &qinq_intf1);

	  identify_subint (hi0,
			   b0,
			   match_flags0,
			   main_intf0,
			   vlan_intf0,
			   qinq_intf0, &new_sw_if_index0, &error0, &is_l20);

	  identify_subint (hi1,
			   b1,
			   match_flags1,
			   main_intf1,
			   vlan_intf1,
			   qinq_intf1, &new_sw_if_index1, &error1, &is_l21);

	  // Save RX sw_if_index for later nodes
	  vnet_buffer (b0)->sw_if_index[VLIB_RX] =
	    error0 !=
	    ETHERNET_ERROR_NONE ? old_sw_if_index0 : new_sw_if_index0;
	  vnet_buffer (b1)->sw_if_index[VLIB_RX] =
	    error1 !=
	    ETHERNET_ERROR_NONE ? old_sw_if_index1 : new_sw_if_index1;

	  // Check if there is a stat to take (valid and non-main sw_if_index for pkt 0 or pkt 1)
	  if (((new_sw_if_index0 != ~0)
	       && (new_sw_if_index0 != old_sw_if_index0))
	      || ((new_sw_if_index1 != ~0)
		  && (new_sw_if_index1 != old_sw_if_index1)))
	    {

	      len0 = vlib_buffer_length_in_chain (vm, b0) + b0->current_data
		- vnet_buffer (b0)->l2_hdr_offset;
	      len1 = vlib_buffer_length_in_chain (vm, b1) + b1->current_data
		- vnet_buffer (b1)->l2_hdr_offset;

	      stats_n_packets += 2;
	      stats_n_bytes += len0 + len1;

	      if (PREDICT_FALSE
		  (!(new_sw_if_index0 == stats_sw_if_index
		     && new_sw_if_index1 == stats_sw_if_index)))
		{
		  stats_n_packets -= 2;
		  stats_n_bytes -= len0 + len1;

		  if (new_sw_if_index0 != old_sw_if_index0
		      && new_sw_if_index0 != ~0)
		    vlib_increment_combined_counter (vnm->
						     interface_main.combined_sw_if_counters
						     +
						     VNET_INTERFACE_COUNTER_RX,
						     thread_index,
						     new_sw_if_index0, 1,
						     len0);
		  if (new_sw_if_index1 != old_sw_if_index1
		      && new_sw_if_index1 != ~0)
		    vlib_increment_combined_counter (vnm->
						     interface_main.combined_sw_if_counters
						     +
						     VNET_INTERFACE_COUNTER_RX,
						     thread_index,
						     new_sw_if_index1, 1,
						     len1);

		  if (new_sw_if_index0 == new_sw_if_index1)
		    {
		      if (stats_n_packets > 0)
			{
			  vlib_increment_combined_counter
			    (vnm->interface_main.combined_sw_if_counters
			     + VNET_INTERFACE_COUNTER_RX,
			     thread_index,
			     stats_sw_if_index,
			     stats_n_packets, stats_n_bytes);
			  stats_n_packets = stats_n_bytes = 0;
			}
		      stats_sw_if_index = new_sw_if_index0;
		    }
		}
	    }

	  if (variant == ETHERNET_INPUT_VARIANT_NOT_L2)
	    is_l20 = is_l21 = 0;

	  determine_next_node (em, variant, is_l20, type0, b0, &error0,
			       &next0);
	  determine_next_node (em, variant, is_l21, type1, b1, &error1,
			       &next1);

	ship_it01:
	  b0->error = error_node->errors[error0];
	  b1->error = error_node->errors[error1];

	  // verify speculative enqueue
	  vlib_validate_buffer_enqueue_x2 (vm, node, next_index, to_next,
					   n_left_to_next, bi0, bi1, next0,
					   next1);
	}

      while (n_left_from > 0 && n_left_to_next > 0)
	{
	  u32 bi0;
	  vlib_buffer_t *b0;
	  u8 error0, next0;
	  u16 type0, orig_type0;
	  u16 outer_id0, inner_id0;
	  u32 match_flags0;
	  u32 old_sw_if_index0, new_sw_if_index0, len0;
	  vnet_hw_interface_t *hi0;
	  main_intf_t *main_intf0;
	  vlan_intf_t *vlan_intf0;
	  qinq_intf_t *qinq_intf0;
	  ethernet_header_t *e0;
	  u32 is_l20;
	  u64 dmacs[2];
	  u8 dmacs_bad[2];

	  // Prefetch next iteration
	  if (n_left_from > 1)
	    {
	      vlib_prefetch_buffer_header (b[1], STORE);
	      CLIB_PREFETCH (b[1]->data, CLIB_CACHE_LINE_BYTES, LOAD);
	    }

	  bi0 = from[0];
	  to_next[0] = bi0;
	  from += 1;
	  to_next += 1;
	  n_left_from -= 1;
	  n_left_to_next -= 1;

	  b0 = b[0];
	  b += 1;

	  error0 = ETHERNET_ERROR_NONE;
	  e0 = vlib_buffer_get_current (b0);
	  type0 = clib_net_to_host_u16 (e0->type);

	  /* Set the L2 header offset for all packets */
	  vnet_buffer (b0)->l2_hdr_offset = b0->current_data;
	  b0->flags |= VNET_BUFFER_F_L2_HDR_OFFSET_VALID;

	  /* Speed-path for the untagged case */
	  if (PREDICT_TRUE (variant == ETHERNET_INPUT_VARIANT_ETHERNET
			    && !ethernet_frame_is_tagged (type0)))
	    {
	      main_intf_t *intf0;
	      subint_config_t *subint0;
	      u32 sw_if_index0;

	      sw_if_index0 = vnet_buffer (b0)->sw_if_index[VLIB_RX];
	      is_l20 = cached_is_l2;

	      if (PREDICT_FALSE (cached_sw_if_index != sw_if_index0))
		{
		  cached_sw_if_index = sw_if_index0;
		  hi = vnet_get_sup_hw_interface (vnm, sw_if_index0);
		  ei = ethernet_get_interface (em, hi->hw_if_index);
		  intf0 = vec_elt_at_index (em->main_intfs, hi->hw_if_index);
		  subint0 = &intf0->untagged_subint;
		  cached_is_l2 = is_l20 = subint0->flags & SUBINT_CONFIG_L2;
		}


	      if (PREDICT_TRUE (is_l20 != 0))
		{
		  vnet_buffer (b0)->l3_hdr_offset =
		    vnet_buffer (b0)->l2_hdr_offset +
		    sizeof (ethernet_header_t);
		  b0->flags |= VNET_BUFFER_F_L3_HDR_OFFSET_VALID;
		  next0 = em->l2_next;
		  vnet_buffer (b0)->l2.l2_len = sizeof (ethernet_header_t);
		}
	      else
		{
		  dmacs[0] = *(u64 *) e0;

		  if (ei && vec_len (ei->secondary_addrs))
		    ethernet_input_inline_dmac_check (hi, dmacs,
						      dmacs_bad,
						      1 /* n_packets */ ,
						      ei,
						      1 /* have_sec_dmac */ );
		  else
		    ethernet_input_inline_dmac_check (hi, dmacs,
						      dmacs_bad,
						      1 /* n_packets */ ,
						      ei,
						      0 /* have_sec_dmac */ );

		  if (dmacs_bad[0])
		    error0 = ETHERNET_ERROR_L3_MAC_MISMATCH;

		  vlib_buffer_advance (b0, sizeof (ethernet_header_t));
		  determine_next_node (em, variant, 0, type0, b0,
				       &error0, &next0);
		}
	      goto ship_it0;
	    }

	  /* Slow-path for the tagged case */
	  parse_header (variant,
			b0,
			&type0,
			&orig_type0, &outer_id0, &inner_id0, &match_flags0);

	  old_sw_if_index0 = vnet_buffer (b0)->sw_if_index[VLIB_RX];

	  eth_vlan_table_lookups (em,
				  vnm,
				  old_sw_if_index0,
				  orig_type0,
				  outer_id0,
				  inner_id0,
				  &hi0,
				  &main_intf0, &vlan_intf0, &qinq_intf0);

	  identify_subint (hi0,
			   b0,
			   match_flags0,
			   main_intf0,
			   vlan_intf0,
			   qinq_intf0, &new_sw_if_index0, &error0, &is_l20);

	  // Save RX sw_if_index for later nodes
	  vnet_buffer (b0)->sw_if_index[VLIB_RX] =
	    error0 !=
	    ETHERNET_ERROR_NONE ? old_sw_if_index0 : new_sw_if_index0;

	  // Increment subinterface stats
	  // Note that interface-level counters have already been incremented
	  // prior to calling this function. Thus only subinterface counters
	  // are incremented here.
	  //
	  // Interface level counters include packets received on the main
	  // interface and all subinterfaces. Subinterface level counters
	  // include only those packets received on that subinterface
	  // Increment stats if the subint is valid and it is not the main intf
	  if ((new_sw_if_index0 != ~0)
	      && (new_sw_if_index0 != old_sw_if_index0))
	    {

	      len0 = vlib_buffer_length_in_chain (vm, b0) + b0->current_data
		- vnet_buffer (b0)->l2_hdr_offset;

	      stats_n_packets += 1;
	      stats_n_bytes += len0;

	      // Batch stat increments from the same subinterface so counters
	      // don't need to be incremented for every packet.
	      if (PREDICT_FALSE (new_sw_if_index0 != stats_sw_if_index))
		{
		  stats_n_packets -= 1;
		  stats_n_bytes -= len0;

		  if (new_sw_if_index0 != ~0)
		    vlib_increment_combined_counter
		      (vnm->interface_main.combined_sw_if_counters
		       + VNET_INTERFACE_COUNTER_RX,
		       thread_index, new_sw_if_index0, 1, len0);
		  if (stats_n_packets > 0)
		    {
		      vlib_increment_combined_counter
			(vnm->interface_main.combined_sw_if_counters
			 + VNET_INTERFACE_COUNTER_RX,
			 thread_index,
			 stats_sw_if_index, stats_n_packets, stats_n_bytes);
		      stats_n_packets = stats_n_bytes = 0;
		    }
		  stats_sw_if_index = new_sw_if_index0;
		}
	    }

	  if (variant == ETHERNET_INPUT_VARIANT_NOT_L2)
	    is_l20 = 0;

	  determine_next_node (em, variant, is_l20, type0, b0, &error0,
			       &next0);

	ship_it0:
	  b0->error = error_node->errors[error0];

	  // verify speculative enqueue
	  vlib_validate_buffer_enqueue_x1 (vm, node, next_index,
					   to_next, n_left_to_next,
					   bi0, next0);
	}

      vlib_put_next_frame (vm, node, next_index, n_left_to_next);
    }

  // Increment any remaining batched stats
  if (stats_n_packets > 0)
    {
      vlib_increment_combined_counter
	(vnm->interface_main.combined_sw_if_counters
	 + VNET_INTERFACE_COUNTER_RX,
	 thread_index, stats_sw_if_index, stats_n_packets, stats_n_bytes);
      node->runtime_data[0] = stats_sw_if_index;
    }
}
```