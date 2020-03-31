## L2

### 处理

#### vnet/l2/l2_input.c

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
### 配置

#### vnet/l2/l2_bd.c

#### vnet/l2/l2_fib.c

**添加**
```
VLIB_CLI_COMMAND (l2fib_add_cli, static) = {
  .path = "l2fib add",
  .short_help = "l2fib add <mac> <bridge-domain-id> filter | <intf> [static | bvi]",
  .function = l2fib_add,
};

/**
 * Add an entry to the L2FIB.
 * The CLI format is:
 *    l2fib add <mac> <bd> <intf> [static] [bvi]
 *    l2fib add <mac> <bd> filter
 * Note that filter and bvi entries are always static
 */
static clib_error_t *
l2fib_add (vlib_main_t * vm,
	   unformat_input_t * input, vlib_cli_command_t * cmd)
{
  bd_main_t *bdm = &bd_main;
  vnet_main_t *vnm = vnet_get_main ();
  clib_error_t *error = 0;
  u8 mac[6];
  u32 bd_id;
  u32 bd_index;
  u32 sw_if_index = ~0;
  uword *p;
  l2fib_entry_result_flags_t flags;

  /* L2 fib entry's flag */
  flags = L2FIB_ENTRY_RESULT_FLAG_NONE;

  /* 解析mac地址 */
  if (!unformat (input, "%U", unformat_ethernet_address, mac))
  {
    error = clib_error_return (0, "expected mac address `%U'", format_unformat_error, input);
    goto done;
  }

  /* 解析bd id */
  if (!unformat (input, "%d", &bd_id))
  {
    error = clib_error_return (0, "expected bridge domain ID `%U'", format_unformat_error, input);
    goto done;
  }

  /* 根据bd id查找bd索引 */
  p = hash_get (bdm->bd_index_by_bd_id, bd_id);
  if (!p)
  {
    error = clib_error_return (0, "bridge domain ID %d invalid", bd_id);
    goto done;
  }
  bd_index = p[0];

  /* 解析filter */
  if (unformat (input, "filter"))
  {
    /* filter添加到l2fib filter entry中 */
    l2fib_add_filter_entry (mac, bd_index);
    return 0;
  }

  /* 解析interface */
  if (!unformat_user (input, unformat_vnet_sw_interface, vnm, &sw_if_index))
  {
    error = clib_error_return (0, "unknown interface `%U'", format_unformat_error, input);
    goto done;
  }

  /* 静态mac */
  if (unformat (input, "static"))
    flags |= L2FIB_ENTRY_RESULT_FLAG_STATIC;
  /* mac是bvi接口的 */
  else if (unformat (input, "bvi"))
    flags |= (L2FIB_ENTRY_RESULT_FLAG_STATIC | L2FIB_ENTRY_RESULT_FLAG_BVI);

  /* 检查L2模式 */
  if (vec_len (l2input_main.configs) <= sw_if_index)
  {
    error = clib_error_return (0, "Interface sw_if_index %d not in L2 mode", sw_if_index);
    goto done;
  }

  /* 记录添加到l2fib中, (mac, bd_index)=>(sw_if_index, flags) */
  l2fib_add_entry (mac, bd_index, sw_if_index, flags);

done:
  return error;
}

/**
 * Add an entry to the l2fib.
 * If the entry already exists then overwrite it
 */
void
l2fib_add_entry (const u8 * mac, u32 bd_index,
		 u32 sw_if_index, l2fib_entry_result_flags_t flags)
{
  /* l2fib记录key */
  l2fib_entry_key_t key;
  
  l2fib_entry_result_t result;
  __attribute__ ((unused)) u32 bucket_contents;
  l2fib_main_t *fm = &l2fib_main;
  l2learn_main_t *lm = &l2learn_main;
  /* 创建bihash kv */
  BVT (clib_bihash_kv) kv;

  /* set up key, key由mac和bd_index构成 */
  key.raw = l2fib_make_key (mac, bd_index);
  /* 给kv赋key */
  kv.key = key.raw;

  /* check if entry already exist */
  if (BV (clib_bihash_search) (&fm->mac_table, &kv, &kv))
  {
    /* decrement counter if overwriting a learned mac  */
    result.raw = kv.value;
	/* 查找到的记录是dynamic的（允许老化），则重新覆盖该记录，同时由于是覆盖所以不占用计数（后期添加后age scan会计数+1），计数减1 */
    if ((!l2fib_entry_result_is_set_AGE_NOT (&result)) && (lm->global_learn_count))
		lm->global_learn_count--;
  }

  /* set up result */
  result.raw = 0;		/* clear all fields */
  /* raw和fields是联合体 */
  result.fields.sw_if_index = sw_if_index;
  result.fields.flags = flags;

  /* no aging for provisioned entry */
  /* 设置记录static（永不老化） */
  l2fib_entry_result_set_AGE_NOT (&result);

  /* 给kv赋value */
  kv.value = result.raw;

  /* kv加入mac table */
  BV (clib_bihash_add_del) (&fm->mac_table, &kv, 1 /* is_add */ );
}
```

**删除**
```
VLIB_CLI_COMMAND (l2fib_del_cli, static) = {
  .path = "l2fib del",
  .short_help = "l2fib del <mac> <bridge-domain-id> []",
  .function = l2fib_del,
};

/**
 * Delete an entry from the L2FIB.
 * The CLI format is:
 *    l2fib del <mac> <bd-id>
 */
static clib_error_t *
l2fib_del (vlib_main_t * vm,
	   unformat_input_t * input, vlib_cli_command_t * cmd)
{
  bd_main_t *bdm = &bd_main;
  clib_error_t *error = 0;
  u8 mac[6];
  u32 bd_id;
  u32 bd_index;
  uword *p;

  /* 解析mac地址 */
  if (!unformat (input, "%U", unformat_ethernet_address, mac))
  {
    error = clib_error_return (0, "expected mac address `%U'", format_unformat_error, input);
    goto done;
  }

  /* 解析bd id */
  if (!unformat (input, "%d", &bd_id))
  {
    error = clib_error_return (0, "expected bridge domain ID `%U'", format_unformat_error, input);
    goto done;
  }

  /* 根据bd id查找bd索引 */
  p = hash_get (bdm->bd_index_by_bd_id, bd_id);
  if (!p)
  {
    error = clib_error_return (0, "bridge domain ID %d invalid", bd_id);
    goto done;
  }
  bd_index = p[0];

  /* 从l2fib中删除记录 */
  if (l2fib_del_entry (mac, bd_index, 0))
  {
    error = clib_error_return (0, "mac entry not found");
    goto done;
  }

done:
  return error;
}

/**
 * Delete an entry from the l2fib.
 * Return 0 if the entry was deleted, or 1 it was not found or if
 * sw_if_index is non-zero and does not match that in the entry.
 */
u32
l2fib_del_entry (const u8 * mac, u32 bd_index, u32 sw_if_index)
{
  l2fib_entry_result_t result;
  l2fib_main_t *mp = &l2fib_main;
  BVT (clib_bihash_kv) kv;

  /* set up key */
  kv.key = l2fib_make_key (mac, bd_index);

  if (BV (clib_bihash_search) (&mp->mac_table, &kv, &kv))
    return 1;

  result.raw = kv.value;

  /*  check if sw_if_index of entry match */
  if ((sw_if_index != 0) && (sw_if_index != result.fields.sw_if_index))
    return 1;

  /* decrement counter if dynamically learned mac */
  if ((!l2fib_entry_result_is_set_AGE_NOT (&result)) && (l2learn_main.global_learn_count))
    l2learn_main.global_learn_count--;

  /* Remove entry from hash table */
  BV (clib_bihash_add_del) (&mp->mac_table, &kv, 0 /* is_add */ );
  return 0;
}
```

**查看**
```
VLIB_CLI_COMMAND (show_l2fib_cli, static) = {
  .path = "show l2fib",
  .short_help = "show l2fib [all] | [bd_id <nn> | bd_index <nn>] [learn | add] | [raw]",
  .function = show_l2fib,
};

/** Display the contents of the l2fib. */
static clib_error_t *
show_l2fib (vlib_main_t * vm,
	    unformat_input_t * input, vlib_cli_command_t * cmd)
{
  bd_main_t *bdm = &bd_main;
  l2fib_main_t *msm = &l2fib_main;
  u8 raw = 0;
  u32 bd_id;
  l2fib_show_walk_ctx_t ctx = {
    .first_entry = 1,
    .bd_index = ~0,
    .now = (u8) (vlib_time_now (vm) / 60),
    .vm = vm,
    .vnm = msm->vnet_main,
  };

  while (unformat_check_input (input) != UNFORMAT_END_OF_INPUT)
  {
    /* 解析raw标志，即原始信息 */
    if (unformat (input, "raw"))
	{
	  raw = 1;
	  ctx.verbose = 0;
	  break;
	}
	/* 解析verbose标志，即显示详情 */
    else if (unformat (input, "verbose"))
  	  ctx.verbose = 1;
	/* 解析all标志，显示所有bd的l2fib详情 */
    else if (unformat (input, "all"))
	  ctx.verbose = 1;
	/* 解析bd索引标志，显示相应bd的l2fib详情 */
    else if (unformat (input, "bd_index %d", &ctx.bd_index))
	  ctx.verbose = 1;
	/* 显示学习到的l2fib */
    else if (unformat (input, "learn"))
	{
	  ctx.add = 0;
	  ctx.learn = 1;
	  ctx.verbose = 1;
	}
	/* 显示手动add的l2fib	*/
    else if (unformat (input, "add"))
	{
	  ctx.learn = 0;
	  ctx.add = 1;
	  ctx.verbose = 1;
	}
	/* 解析bd id标志，显示相应bd的l2fib详情 */
    else if (unformat (input, "bd_id %d", &bd_id))
	{
	  /* 根据bd id查找bd index */
	  uword *p = hash_get (bdm->bd_index_by_bd_id, bd_id);
	  if (p)
	  {
	    ctx.verbose = 1;
	    ctx.bd_index = p[0];
	  }
	  else
	    return clib_error_return (0,
				      "bridge domain id %d doesn't exist\n",
				      bd_id);
	}
    else
	  break;
  }

  /* 遍历mac table */
  BV (clib_bihash_foreach_key_value_pair)
    (&msm->mac_table, l2fib_show_walk_cb, &ctx);

  /* 没有记录 */
  if (ctx.total_entries == 0)
    vlib_cli_output (vm, "no l2fib entries");
  /* 有记录 */
  else
  {
    l2learn_main_t *lm = &l2learn_main;
	/* 显示学习到的记录和总记录数，以及上次老化检查的时间 */
    vlib_cli_output (vm, "L2FIB total/learned entries: %d/%d  "
		                 "Last scan time: %.4esec  Learn limit: %d ",
		                 ctx.total_entries, lm->global_learn_count,
		                 msm->age_scan_duration, lm->global_learn_limit);
    /* 显示等待mac学习和老化事件的客户端信息 */
    if (lm->client_pid)
	  vlib_cli_output (vm, "L2MAC events client PID: %d  "
			               "Last e-scan time: %.4esec  Delay: %.2esec  "
			               "Max macs in event: %d",
			               lm->client_pid, msm->evt_scan_duration,
			               msm->event_scan_delay, msm->max_macs_in_event);
  }

  /* 显示原始hash表信息 */
  if (raw)
    vlib_cli_output (vm, "Raw Hash Table:\n%U\n",
		     BV (format_bihash), &msm->mac_table, 1 /* verbose */ );

  return 0;
}
```

**老化**
```
VLIB_REGISTER_NODE (l2fib_mac_age_scanner_process_node) = {
    .function = l2fib_mac_age_scanner_process,
    .type = VLIB_NODE_TYPE_PROCESS,
    .name = "l2fib-mac-age-scanner-process",
};

static uword
l2fib_mac_age_scanner_process (vlib_main_t * vm, vlib_node_runtime_t * rt,
			       vlib_frame_t * f)
{
  uword event_type, *event_data = 0;
  l2fib_main_t *fm = &l2fib_main;
  l2learn_main_t *lm = &l2learn_main;
  bool enabled = 0;
  f64 start_time, next_age_scan_time = CLIB_TIME_MAX;

  while (1)
  {
    /* 有客户端等待事件，则等待事件到来或者超时到来 */
    if (lm->client_pid)
		vlib_process_wait_for_event_or_clock (vm, fm->event_scan_delay);
    /* 正在扫描，等待事件到来或者超时（到下次扫描时间的时间间隔）到来 */
    else if (enabled)
	{
	  f64 t = next_age_scan_time - vlib_time_now (vm);
	  vlib_process_wait_for_event_or_clock (vm, t);
	}
	/* 其他情况，等待事件到来 */
    else
		vlib_process_wait_for_event (vm);

    /* 事件到来，获取事件类型 */
    event_type = vlib_process_get_events (vm, &event_data);
    vec_reset_length (event_data);

    start_time = vlib_time_now (vm);
    enum { SCAN_MAC_AGE, SCAN_MAC_EVENT, SCAN_DISABLE } scan = SCAN_MAC_AGE;

    switch (event_type)
	{
		/* 超时 */
		case ~0:		/* timer expired */
		  /* 有客户端连接，执行事件扫描 */
		  if (lm->client_pid != 0 && start_time < next_age_scan_time)
			scan = SCAN_MAC_EVENT;
		  break;
		
		/* 开始扫描 */
		case L2_MAC_AGE_PROCESS_EVENT_START:
		  enabled = 1;
		  break;
		
		/* 停止扫描 */
		case L2_MAC_AGE_PROCESS_EVENT_STOP:
		  enabled = 0;
		  scan = SCAN_DISABLE;
		  break;
		
		/* 过一次 */
		case L2_MAC_AGE_PROCESS_EVENT_ONE_PASS:
		  break;

		default:
		  ASSERT (0);
	}

    if (scan == SCAN_MAC_EVENT)
	    /* 时间扫描 */
		l2fib_main.evt_scan_duration = l2fib_scan (vm, start_time, 1);
    else
	{
	  /* 老化扫描 */
	  if (scan == SCAN_MAC_AGE)
	    l2fib_main.age_scan_duration = l2fib_scan (vm, start_time, 0);
	  /* 扫描禁用 */
	  if (scan == SCAN_DISABLE)
	  {
	    l2fib_main.age_scan_duration = 0;
	    l2fib_main.evt_scan_duration = 0;
	  }
	  /* schedule next scan */
	  if (enabled)
	    next_age_scan_time = start_time + L2FIB_AGE_SCAN_INTERVAL;
	  else
	    next_age_scan_time = CLIB_TIME_MAX;
	}
  }
  return 0;
}

static_always_inline f64
l2fib_scan (vlib_main_t * vm, f64 start_time, u8 event_only)
{
  l2fib_main_t *fm = &l2fib_main;
  l2learn_main_t *lm = &l2learn_main;

  BVT (clib_bihash) * h = &fm->mac_table;
  int i, j, k;
  f64 last_start = start_time;
  f64 accum_t = 0;
  f64 delta_t = 0;
  u32 evt_idx = 0;
  u32 learn_count = 0;
  u32 client = lm->client_pid;
  u32 cl_idx = lm->client_index;
  vl_api_l2_macs_event_t *mp = 0;
  vl_api_registration_t *reg = 0;

  /* Don't scan the l2 fib if it hasn't been instantiated yet */
  if (alloc_arena (h) == 0)
    return 0.0;

  if (client)
    {
      mp = allocate_mac_evt_buf (client, cl_idx);
      reg = vl_api_client_index_to_registration (lm->client_index);
    }

  for (i = 0; i < h->nbuckets; i++)
    {
      /* allow no more than 20us without a pause */
      delta_t = vlib_time_now (vm) - last_start;
      if (delta_t > 20e-6)
	{
	  vlib_process_suspend (vm, 100e-6);	/* suspend for 100 us */
	  last_start = vlib_time_now (vm);
	  accum_t += delta_t;
	}

      if (i < (h->nbuckets - 3))
	{
	  BVT (clib_bihash_bucket) * b = &h->buckets[i + 3];
	  CLIB_PREFETCH (b, CLIB_CACHE_LINE_BYTES, LOAD);
	  b = &h->buckets[i + 1];
	  if (b->offset)
	    {
	      BVT (clib_bihash_value) * v =
		BV (clib_bihash_get_value) (h, b->offset);
	      CLIB_PREFETCH (v, CLIB_CACHE_LINE_BYTES, LOAD);
	    }
	}

      BVT (clib_bihash_bucket) * b = &h->buckets[i];
      if (b->offset == 0)
	continue;
      BVT (clib_bihash_value) * v = BV (clib_bihash_get_value) (h, b->offset);
      for (j = 0; j < (1 << b->log2_pages); j++)
	{
	  for (k = 0; k < BIHASH_KVP_PER_PAGE; k++)
	    {
	      if (v->kvp[k].key == ~0ULL && v->kvp[k].value == ~0ULL)
		continue;

	      l2fib_entry_key_t key = {.raw = v->kvp[k].key };
	      l2fib_entry_result_t result = {.raw = v->kvp[k].value };

	      if (!l2fib_entry_result_is_set_AGE_NOT (&result))
		learn_count++;

	      if (client)
		{
		  if (PREDICT_FALSE (evt_idx >= fm->max_macs_in_event))
		    {
		      /* event message full, send it and start a new one */
		      if (reg && vl_api_can_send_msg (reg))
			{
			  mp->n_macs = htonl (evt_idx);
			  vl_api_send_msg (reg, (u8 *) mp);
			  mp = allocate_mac_evt_buf (client, cl_idx);
			}
		      else
			{
			  if (reg)
			    clib_warning ("MAC event to pid %d queue stuffed!"
					  " %d MAC entries lost", client,
					  evt_idx);
			}
		      evt_idx = 0;
		    }

		  if (l2fib_entry_result_is_set_LRN_EVT (&result))
		    {
		      /* copy mac entry to event msg */
		      clib_memcpy_fast (mp->mac[evt_idx].mac_addr,
					key.fields.mac, 6);
		      mp->mac[evt_idx].action =
			l2fib_entry_result_is_set_LRN_MOV (&result) ?
			(vl_api_mac_event_action_t) MAC_EVENT_ACTION_MOVE
			: (vl_api_mac_event_action_t) MAC_EVENT_ACTION_ADD;
		      mp->mac[evt_idx].action =
			htonl (mp->mac[evt_idx].action);
		      mp->mac[evt_idx].sw_if_index =
			htonl (result.fields.sw_if_index);
		      /* clear event bits and update mac entry */
		      l2fib_entry_result_clear_LRN_EVT (&result);
		      l2fib_entry_result_clear_LRN_MOV (&result);
		      BVT (clib_bihash_kv) kv;
		      kv.key = key.raw;
		      kv.value = result.raw;
		      BV (clib_bihash_add_del) (&fm->mac_table, &kv, 1);
		      evt_idx++;
		      continue;	/* skip aging */
		    }
		}

	      if (event_only || l2fib_entry_result_is_set_AGE_NOT (&result))
		continue;	/* skip aging - static_mac always age_not */

	      /* start aging processing */
	      u32 bd_index = key.fields.bd_index;
	      u32 sw_if_index = result.fields.sw_if_index;
	      u16 sn = l2fib_cur_seq_num (bd_index, sw_if_index).as_u16;
	      if (result.fields.sn.as_u16 != sn)
		goto age_out;	/* stale mac */

	      l2_bridge_domain_t *bd_config =
		vec_elt_at_index (l2input_main.bd_configs, bd_index);

	      if (bd_config->mac_age == 0)
		continue;	/* skip aging */

	      i16 delta = (u8) (start_time / 60) - result.fields.timestamp;
	      delta += delta < 0 ? 256 : 0;

	      if (delta < bd_config->mac_age)
		continue;	/* still valid */

	    age_out:
	      if (client)
		{
		  /* copy mac entry to event msg */
		  clib_memcpy_fast (mp->mac[evt_idx].mac_addr, key.fields.mac,
				    6);
		  mp->mac[evt_idx].action =
		    (vl_api_mac_event_action_t) MAC_EVENT_ACTION_DELETE;
		  mp->mac[evt_idx].action = htonl (mp->mac[evt_idx].action);
		  mp->mac[evt_idx].sw_if_index =
		    htonl (result.fields.sw_if_index);
		  evt_idx++;
		}
	      /* delete mac entry */
	      BVT (clib_bihash_kv) kv;
	      kv.key = key.raw;
	      BV (clib_bihash_add_del) (&fm->mac_table, &kv, 0);
	      learn_count--;
	      /*
	       * Note: we may have just freed the bucket's backing
	       * storage, so check right here...
	       */
	      if (b->offset == 0)
		goto doublebreak;
	    }
	  v++;
	}
    doublebreak:
      ;
    }

  /* keep learn count consistent */
  l2learn_main.global_learn_count = learn_count;

  if (mp)
    {
      /*  send any outstanding mac event message else free message buffer */
      if (evt_idx)
	{
	  if (reg && vl_api_can_send_msg (reg))
	    {
	      mp->n_macs = htonl (evt_idx);
	      vl_api_send_msg (reg, (u8 *) mp);
	    }
	  else
	    {
	      if (reg)
		clib_warning ("MAC event to pid %d queue stuffed!"
			      " %d MAC entries lost", client, evt_idx);
	      vl_msg_api_free (mp);
	    }
	}
      else
	vl_msg_api_free (mp);
    }
  return delta_t + accum_t;
}
```