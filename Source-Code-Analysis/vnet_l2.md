## L2

### 处理

### vnet/l2/l2_input.c

```
VLIB_REGISTER_NODE (l2input_node) = {
  .name = "l2-input",
  .vector_size = sizeof (u32),
  .format_trace = format_l2input_trace,
  .format_buffer = format_ethernet_header_with_length,
  .type = VLIB_NODE_TYPE_INTERNAL,

  .n_errors = ARRAY_LEN(l2input_error_strings),
  .error_strings = l2input_error_strings,

  .n_next_nodes = L2INPUT_N_NEXT,

  /* edit / add dispositions here */
  .next_nodes = {
       [L2INPUT_NEXT_LEARN] = "l2-learn",
       [L2INPUT_NEXT_FWD]   = "l2-fwd",
       [L2INPUT_NEXT_DROP]  = "error-drop",
  },
};

VLIB_NODE_FN (l2input_node) (vlib_main_t * vm,
                 vlib_node_runtime_t * node, vlib_frame_t * frame)
{
  if (PREDICT_FALSE ((node->flags & VLIB_NODE_FLAG_TRACE)))
    return l2input_node_inline (vm, node, frame, 1 /* do_trace */ );
  return l2input_node_inline (vm, node, frame, 0 /* do_trace */ );
}
```

```
static_always_inline uword
l2input_node_inline (vlib_main_t * vm,
             vlib_node_runtime_t * node, vlib_frame_t * frame,
             int do_trace)
{
  u32 n_left_from, *from, *to_next;
  l2input_next_t next_index;
  l2input_main_t *msm = &l2input_main;

  from = vlib_frame_vector_args (frame);
  n_left_from = frame->n_vectors;    /* number of packets to process */
  next_index = node->cached_next_index;

  while (n_left_from > 0)
    {
      u32 n_left_to_next;

      /* get space to enqueue frame to graph node "next_index" */
      vlib_get_next_frame (vm, node, next_index, to_next, n_left_to_next);

      while (n_left_from >= 8 && n_left_to_next >= 4)
    {
      u32 bi0, bi1, bi2, bi3;
      vlib_buffer_t *b0, *b1, *b2, *b3;
      u32 next0, next1, next2, next3;
      u32 sw_if_index0, sw_if_index1, sw_if_index2, sw_if_index3;

      /* Prefetch next iteration. */
      {
        vlib_buffer_t *p4, *p5, *p6, *p7;

        p4 = vlib_get_buffer (vm, from[4]);
        p5 = vlib_get_buffer (vm, from[5]);
        p6 = vlib_get_buffer (vm, from[6]);
        p7 = vlib_get_buffer (vm, from[7]);

        /* Prefetch the buffer header and packet for the N+2 loop iteration */
        vlib_prefetch_buffer_header (p4, LOAD);
        vlib_prefetch_buffer_header (p5, LOAD);
        vlib_prefetch_buffer_header (p6, LOAD);
        vlib_prefetch_buffer_header (p7, LOAD);

        CLIB_PREFETCH (p4->data, CLIB_CACHE_LINE_BYTES, STORE);
        CLIB_PREFETCH (p5->data, CLIB_CACHE_LINE_BYTES, STORE);
        CLIB_PREFETCH (p6->data, CLIB_CACHE_LINE_BYTES, STORE);
        CLIB_PREFETCH (p7->data, CLIB_CACHE_LINE_BYTES, STORE);

        /*
         * Don't bother prefetching the bridge-domain config (which
         * depends on the input config above). Only a small number of
         * bridge domains are expected. Plus the structure is small
         * and several fit in a cache line.
         */
      }

      /* speculatively enqueue b0 and b1 to the current next frame */
      /* bi is "buffer index", b is pointer to the buffer */
      to_next[0] = bi0 = from[0];
      to_next[1] = bi1 = from[1];
      to_next[2] = bi2 = from[2];
      to_next[3] = bi3 = from[3];
      from += 4;
      to_next += 4;
      n_left_from -= 4;
      n_left_to_next -= 4;

      b0 = vlib_get_buffer (vm, bi0);
      b1 = vlib_get_buffer (vm, bi1);
      b2 = vlib_get_buffer (vm, bi2);
      b3 = vlib_get_buffer (vm, bi3);

      if (do_trace)
        {
          /* RX interface handles */
          sw_if_index0 = vnet_buffer (b0)->sw_if_index[VLIB_RX];
          sw_if_index1 = vnet_buffer (b1)->sw_if_index[VLIB_RX];
          sw_if_index2 = vnet_buffer (b2)->sw_if_index[VLIB_RX];
          sw_if_index3 = vnet_buffer (b3)->sw_if_index[VLIB_RX];

          if (b0->flags & VLIB_BUFFER_IS_TRACED)
        {
          ethernet_header_t *h0 = vlib_buffer_get_current (b0);
          l2input_trace_t *t =
            vlib_add_trace (vm, node, b0, sizeof (*t));
          t->sw_if_index = sw_if_index0;
          clib_memcpy_fast (t->src, h0->src_address, 6);
          clib_memcpy_fast (t->dst, h0->dst_address, 6);
        }
          if (b1->flags & VLIB_BUFFER_IS_TRACED)
        {
          ethernet_header_t *h1 = vlib_buffer_get_current (b1);
          l2input_trace_t *t =
            vlib_add_trace (vm, node, b1, sizeof (*t));
          t->sw_if_index = sw_if_index1;
          clib_memcpy_fast (t->src, h1->src_address, 6);
          clib_memcpy_fast (t->dst, h1->dst_address, 6);
        }
          if (b2->flags & VLIB_BUFFER_IS_TRACED)
        {
          ethernet_header_t *h2 = vlib_buffer_get_current (b2);
          l2input_trace_t *t =
            vlib_add_trace (vm, node, b2, sizeof (*t));
          t->sw_if_index = sw_if_index2;
          clib_memcpy_fast (t->src, h2->src_address, 6);
          clib_memcpy_fast (t->dst, h2->dst_address, 6);
        }
          if (b3->flags & VLIB_BUFFER_IS_TRACED)
        {
          ethernet_header_t *h3 = vlib_buffer_get_current (b3);
          l2input_trace_t *t =
            vlib_add_trace (vm, node, b3, sizeof (*t));
          t->sw_if_index = sw_if_index3;
          clib_memcpy_fast (t->src, h3->src_address, 6);
          clib_memcpy_fast (t->dst, h3->dst_address, 6);
        }
        }

      /* 核心函数：分类并调度，确定next节点 */
      classify_and_dispatch (msm, b0, &next0);
      classify_and_dispatch (msm, b1, &next1);
      classify_and_dispatch (msm, b2, &next2);
      classify_and_dispatch (msm, b3, &next3);

      /* verify speculative enqueues, maybe switch current next frame */
      /* if next0==next1==next_index then nothing special needs to be done */
      vlib_validate_buffer_enqueue_x4 (vm, node, next_index,
                       to_next, n_left_to_next,
                       bi0, bi1, bi2, bi3,
                       next0, next1, next2, next3);
    }

      while (n_left_from > 0 && n_left_to_next > 0)
    {
      u32 bi0;
      vlib_buffer_t *b0;
      u32 next0;
      u32 sw_if_index0;

      /* speculatively enqueue b0 to the current next frame */
      bi0 = from[0];
      to_next[0] = bi0;
      from += 1;
      to_next += 1;
      n_left_from -= 1;
      n_left_to_next -= 1;

      b0 = vlib_get_buffer (vm, bi0);

      if (do_trace && PREDICT_FALSE (b0->flags & VLIB_BUFFER_IS_TRACED))
        {
          ethernet_header_t *h0 = vlib_buffer_get_current (b0);
          l2input_trace_t *t = vlib_add_trace (vm, node, b0, sizeof (*t));
          sw_if_index0 = vnet_buffer (b0)->sw_if_index[VLIB_RX];
          t->sw_if_index = sw_if_index0;
          clib_memcpy_fast (t->src, h0->src_address, 6);
          clib_memcpy_fast (t->dst, h0->dst_address, 6);
        }

      classify_and_dispatch (msm, b0, &next0);

      /* verify speculative enqueue, maybe switch current next frame */
      vlib_validate_buffer_enqueue_x1 (vm, node, next_index,
                       to_next, n_left_to_next,
                       bi0, next0);
    }

      vlib_put_next_frame (vm, node, next_index, n_left_to_next);
    }

  vlib_node_increment_counter (vm, l2input_node.index,
                   L2INPUT_ERROR_L2INPUT, frame->n_vectors);

  return frame->n_vectors;
}
```

```
static_always_inline void
classify_and_dispatch (l2input_main_t * msm, vlib_buffer_t * b0, u32 * next0)
{
  /*
   * Load L2 input feature struct
   * Load bridge domain struct
   * Parse ethernet header to determine unicast/mcast/broadcast
   * take L2 input stat
   * classify packet as IP/UDP/TCP, control, other
   * mask feature bitmap
   * go to first node in bitmap
   * Later: optimize VTM
   *
   * For L2XC,
   *   set tx sw-if-handle
   */

  /* 针对此数据包的feature掩码 */
  u32 feat_mask = ~0;
  u32 sw_if_index0 = vnet_buffer (b0)->sw_if_index[VLIB_RX];
  ethernet_header_t *h0 = vlib_buffer_get_current (b0);

  /* Get config for the input interface */
  l2_input_config_t *config = vec_elt_at_index (msm->configs, sw_if_index0);

  /* Save split horizon group */
  /* 水平分割（split horizon）是指，从一端收到的路由信息，不能再从原路被发送回去。为了路由环路。 */
  /* 根据接口配置，设置水平分隔组号（0关闭，非0位组号） */
  vnet_buffer (b0)->l2.shg = config->shg;

  /* determine layer2 kind for stat and mask */
  /* 检查dest mac地址的i/g位，I/G(Individual/Group)位,如果I/G=0,则是某台设备的MAC地址,即单播地址;如果I/G=1,则是多播地址(组播+广播=多播) */
  /* 单播，unicast */
  if (PREDICT_FALSE (ethernet_address_cast (h0->dst_address)))
  {
      /* L3头 */
      u8 *l3h0 = (u8 *) h0 + vnet_buffer (b0)->l2.l2_len;

#define get_u16(addr) ( *((u16 *)(addr)) )
      /* 以太网类型 */
      u16 ethertype = clib_net_to_host_u16 (get_u16 (l3h0 - 2));
      /* ip上层协议 */
      u8 protocol = ((ip6_header_t *) l3h0)->protocol;

      /* Disable bridge forwarding (flooding will execute instead if not xconnect) */
      /* 
       * L2INPUT_FEAT_FWD: Forwarding
       * L2INPUT_FEAT_UU_FWD: Unknown Unicast Forwarding
       * L2INPUT_FEAT_UU_FLOOD: Unknown Unicast Flooding
       * L2INPUT_FEAT_GBP_FWD: GBP Forwarding
       * 初始feature掩码：禁用转发forwarding（但未关闭洪范flooding）
       */
      feat_mask &= ~(L2INPUT_FEAT_FWD | L2INPUT_FEAT_UU_FLOOD | L2INPUT_FEAT_UU_FWD | L2INPUT_FEAT_GBP_FWD);

      /* 不是arp包，则禁用ARP Unicast Forwarding */
      if (ethertype != ETHERNET_TYPE_ARP)
        feat_mask &= ~(L2INPUT_FEAT_ARP_UFWD);

      /* Disable ARP-term for non-ARP and non-ICMP6 packet */
      /* 不是arp包，且不是icmpv6，则禁用ARP-termination来避免洪范arp请求 */
      if (ethertype != ETHERNET_TYPE_ARP && (ethertype != ETHERNET_TYPE_IP6 || protocol != IP_PROTOCOL_ICMP6))
        feat_mask &= ~(L2INPUT_FEAT_ARP_TERM);
      /*
       * For packet from BVI - set SHG of ARP request or ICMPv6 neighbor
       * solicitation packet from BVI to 0 so it can also flood to VXLAN
       * tunnels or other ports with the same SHG as that of the BVI.
       */
      /* 如果数据包出口接口是bvi接口 */
      else if (PREDICT_FALSE (vnet_buffer (b0)->sw_if_index[VLIB_TX] == L2INPUT_BVI))
      {
        /* arp包 */
        if (ethertype == ETHERNET_TYPE_ARP)
        {
          /* arp头 */
          ethernet_arp_header_t *arp0 = (ethernet_arp_header_t *) l3h0;
          /* 是arp请求，则关闭水平分隔，可以继续洪范到vxlan等隧道 */
          if (arp0->opcode == clib_host_to_net_u16 (ETHERNET_ARP_OPCODE_request))
            vnet_buffer (b0)->l2.shg = 0;
        }
        /* icmpv6包 */
        else            /* must be ICMPv6 */
        {
          /* ipv6头 */
          ip6_header_t *iph0 = (ip6_header_t *) l3h0;
          /* nd头 */
          icmp6_neighbor_solicitation_or_advertisement_header_t *ndh0;
          ndh0 = ip6_next_header (iph0);
          /* 是邻居请求，则关闭水平分隔，可以继续洪范到vxlan等隧道 */
          if (ndh0->icmp.type == ICMP6_neighbor_solicitation)
            vnet_buffer (b0)->l2.shg = 0;
        }
      }
  }
  /* 多播（组播+广播） */
  else
  {
      /*
       * For packet from BVI - set SHG of unicast packet from BVI to 0 so it
       * is not dropped on output to VXLAN tunnels or other ports with the
       * same SHG as that of the BVI.
       */
      /* 数据包出口接口是bvi接口，则关闭水平分隔，可以继续洪范到vxlan等隧道 */
      if (PREDICT_FALSE (vnet_buffer (b0)->sw_if_index[VLIB_TX] == L2INPUT_BVI))
        vnet_buffer (b0)->l2.shg = 0;
  }

  /* 配置了网桥 */
  if (config->bridge)
  {
      /* Do bridge-domain processing */
      u16 bd_index0 = config->bd_index;
      /* save BD ID for next feature graph nodes */
      vnet_buffer (b0)->l2.bd_index = bd_index0;

      /* Get config for the bridge domain interface */
      l2_bridge_domain_t *bd_config =
      vec_elt_at_index (msm->bd_configs, bd_index0);

      /* Save bridge domain and interface seq_num */
      /* *INDENT-OFF* */
      l2fib_seq_num_t sn = {
        .swif = *l2fib_swif_seq_num(sw_if_index0),
        .bd = bd_config->seq_num,
      };
      /* *INDENT-ON* */
      vnet_buffer (b0)->l2.l2fib_sn = sn.as_u16;;
      vnet_buffer (b0)->l2.bd_age = bd_config->mac_age;

      /*
       * Process bridge domain feature enables.
       * To perform learning/flooding/forwarding, the corresponding bit
       * must be enabled in both the input interface config and in the
       * bridge domain config. In the bd_bitmap, bits for features other
       * than learning/flooding/forwarding should always be set.
       */
      /* 数据包处理feature掩码增加网桥feature掩码 */
      feat_mask = feat_mask & bd_config->feature_bitmap;
  }
  /* 配置了xconnect */
  else if (config->xconnect)
  {
      /* Set the output interface */
      /* 是xconnect，则数据包输出接口设置为xconnect设置的输出接口 */
      vnet_buffer (b0)->sw_if_index[VLIB_TX] = config->output_sw_if_index;
  }
  /* L3模式，错误，丢弃 */
  else
      feat_mask = L2INPUT_FEAT_DROP;

  /* mask out features from bitmap using packet type and bd config */
  u32 feature_bitmap = config->feature_bitmap & feat_mask;

  /* save for next feature graph nodes */
  vnet_buffer (b0)->l2.feature_bitmap = feature_bitmap;

  /* Determine the next node */
  /* 返回处理首个feature的节点 */
  *next0 = feat_bitmap_get_next_node_index (msm->feat_next_node_index,
                        feature_bitmap);
}
```
