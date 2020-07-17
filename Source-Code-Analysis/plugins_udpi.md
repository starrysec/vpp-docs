## UDPI

udpi是基于vpp的深度数据包检测插件。

### 配置

#### plugins/udpi/dpi_cli.c

**add/delete flow**
```
VLIB_CLI_COMMAND (dpi_flow_add_del_command, static) = {
    .path = "dpi flow",
    .short_help = "dpi flow [add | del] "
        "[src-ip <ip-addr>] [dst-ip <ip-addr>] "
        "[src-port <port>] [dst-port <port>] "
        "[protocol <protocol>] [vrf-id <nn>]",
    .function = dpi_flow_add_del_command_fn,
};

static clib_error_t *
dpi_flow_add_del_command_fn (vlib_main_t * vm,
			     unformat_input_t * input,
			     vlib_cli_command_t * cmd_arg)
{
  unformat_input_t _line_input, *line_input = &_line_input;
  ip46_address_t src_ip = ip46_address_initializer;
  ip46_address_t dst_ip = ip46_address_initializer;
  u16 src_port = 0, dst_port = 0;
  u8 is_add = 0;
  u8 ipv4_set = 0;
  u8 ipv6_set = 0;
  u32 tmp;
  int rv;
  u8 protocol = 0;
  u32 table_id;
  u32 fib_index = 0;
  u32 dpi_flow_id;

  if (!unformat_user (input, unformat_line_input, line_input))
    return 0;

  while (unformat_check_input (line_input) != UNFORMAT_END_OF_INPUT)
  {
	  // action: add
      if (unformat (line_input, "add"))
		is_add = 1;
	  // action: delete
      else if (unformat (line_input, "del"))
		is_add = 0;
	  // parse src-ip field.
      else if (unformat (line_input, "src-ip %U", unformat_ip46_address, &src_ip, IP46_TYPE_ANY))
	  {
	    ip46_address_is_ip4 (&src_ip) ? (ipv4_set = 1) : (ipv6_set = 1);
	  }
	  // parse dst-ip field.
      else if (unformat (line_input, "dst-ip %U", unformat_ip46_address, &dst_ip, IP46_TYPE_ANY))
	  {
	    ip46_address_is_ip4 (&dst_ip) ? (ipv4_set = 1) : (ipv6_set = 1);
	  }
	  // parse src-port field.
      else if (unformat (line_input, "src-port %d", &tmp))
		src_port = (u16) tmp;
	  // parse dst-port field.
      else if (unformat (line_input, "dst-port %d", &tmp))
		dst_port = (u16) tmp;
	  // parse ip-protocol field.
      else if (unformat (line_input, "protocol %U", unformat_ip_protocol, &tmp))
		protocol = (u8) tmp;
      else if (unformat (line_input, "protocol %u", &tmp))
		protocol = (u8) tmp;
	  // parse vrf-id field. VRF: Virtual Routing Forwarding.
      else if (unformat (line_input, "vrf-id %d", &table_id))
	  {
	    // find fib index by table id.
	    fib_index = fib_table_find (fib_ip_proto (ipv6_set), table_id);
	  }
	  // error occurred.
      else
	    return clib_error_return (0, "parse error: '%U'", format_unformat_error, line_input);
  }

  unformat_free (line_input);

  if (ipv4_set && ipv6_set)
    return clib_error_return (0, "both IPv4 and IPv6 addresses specified");

  // fill flow args.
  dpi_add_del_flow_args_t a = {.is_add = is_add,
    .is_ipv6 = ipv6_set,
#define _(x) .x = x,
    // fill src_ip, dst_ip, src_port, dst_port, protocol, and fib_index fields.
    foreach_copy_field
#undef _
  };

  /* Add normal flow, return flow id. */
  rv = dpi_flow_add_del (&a, &dpi_flow_id);
  if (rv < 0)
    return clib_error_return (0, "flow error: %d", rv);

  /* Add reverse flow */
  rv = dpi_reverse_flow_add_del (&a, dpi_flow_id);
  if (rv < 0)
    return clib_error_return (0, "reverse flow error: %d", rv);

  // print flow id.
  vlib_cli_output(vm, "dpi flow id\n%u", dpi_flow_id);

  return 0;
}
```

**tcp reassembly**
```
VLIB_CLI_COMMAND (dpi_tcp_reass_command, static) = {
    .path = "dpi tcp reass",
    .short_help = "dpi tcp reass flow_id <nn> <enable|disable> "
        "[ <client | server | both> ]",
    .function = dpi_tcp_reass_command_fn,
};

static clib_error_t *
dpi_tcp_reass_command_fn (vlib_main_t * vm,
			  unformat_input_t * input,
			  vlib_cli_command_t * cmd_arg)
{
  unformat_input_t _line_input, *line_input = &_line_input;
  u32 flow_id = ~0;
  u8 reass_en = 0;
  u8 reass_dir = 0;
  int rv;

  if (!unformat_user (input, unformat_line_input, line_input))
    return 0;

  while (unformat_check_input (line_input) != UNFORMAT_END_OF_INPUT)
  {
	// parse flow id.
    if (unformat (line_input, "flow_id %d", &flow_id));
	// enable or disable tcp reassembly.
    else if (unformat (line_input, "enable"))
	{
	  reass_en = 1;
	}
    else if (unformat (line_input, "disable"))
	{
	  reass_en = 0;
	}
	// parse tcp reassembly direction. client, server or both.
    else if (unformat (line_input, "client"))
	{
	  reass_dir = REASS_C2S;
	}
    else if (unformat (line_input, "server"))
	{
	  reass_dir = REASS_S2C;
	}
    else if (unformat (line_input, "both"))
	{
	  reass_dir = REASS_BOTH;
	}
	  // error occurred.
      else
	return clib_error_return (0, "parse error: '%U'",
				  format_unformat_error, line_input);
  }

  unformat_free (line_input);

  // fill tcp reassembly args.
  tcp_reass_args_t a = {.flow_id = flow_id,
    .reass_en = reass_en,
    .reass_dir = reass_dir,
  };

  // set flow's reassembly infos.
  rv = dpi_tcp_reass (&a);
  if (rv < 0)
    return clib_error_return (0, "flow error: %d", rv);

  return 0;
}
```

**set flow offload**
```
VLIB_CLI_COMMAND (dpi_flow_offload_command, static) = {
    .path = "dpi set flow-offload",
    .short_help =
        "dpi set flow-offload hw <interface-name> rx <flow-id> [del]",
    .function = dpi_flow_offload_command_fn,
};

static clib_error_t *
dpi_flow_offload_command_fn (vlib_main_t * vm,
			     unformat_input_t * input,
			     vlib_cli_command_t * cmd)
{
  unformat_input_t _line_input, *line_input = &_line_input;
  dpi_main_t *dm = &dpi_main;
  vnet_main_t *vnm = dm->vnet_main;
  u32 rx_flow_id = ~0;
  u32 hw_if_index = ~0;
  int is_add = 1;
  u32 is_ipv6 = 0;
  u32 is_enable = 1;
  dpi_flow_entry_t *flow;
  vnet_hw_interface_t *hw_if;
  u32 rx_fib_index = ~0;

  /* Get a line of input. */
  if (!unformat_user (input, unformat_line_input, line_input))
    return 0;

  while (unformat_check_input (line_input) != UNFORMAT_END_OF_INPUT)
  {
    // parse hardware interface name.
    if (unformat (line_input, "hw %U", unformat_vnet_hw_interface, vnm,
		    &hw_if_index))
	  continue;
	// parse rx flow id.
    if (unformat (line_input, "rx %d", &rx_flow_id))
	  continue;
	// parse action: add or delete.
    if (unformat (line_input, "del"))
	{
	  is_add = 0;
	  continue;
	}
	// error occurred.
    return clib_error_return (0, "unknown input `%U'",
				format_unformat_error, line_input);
  }

  // check flow id and hw interface.
  if (rx_flow_id == ~0)
    return clib_error_return (0, "missing rx flow");
  if (hw_if_index == ~0)
    return clib_error_return (0, "missing hw interface");

  // get flow entry by rx flow id.
  flow = pool_elt_at_index (dm->dpi_flows, rx_flow_id);

  // get hw interface structure by hw interface index.
  hw_if = vnet_get_hw_interface (vnm, hw_if_index);

  // ipv6 or ipv4.
  is_ipv6 = ip46_address_is_ip4 (&(flow->key.src_ip)) ? 0 : 1;

  // get rx fib index by sw interface index.
  if (is_ipv6)
  {
      ip6_main_t *im6 = &ip6_main;
      rx_fib_index = vec_elt (im6->fib_index_by_sw_if_index, hw_if->sw_if_index);
  }
  else
  {
      ip4_main_t *im4 = &ip4_main;
      rx_fib_index = vec_elt (im4->fib_index_by_sw_if_index, hw_if->sw_if_index);
  }

  // check flow fib index is equal to rx fib index.
  if (flow->key.fib_index != rx_fib_index)
    return clib_error_return (0, "interface/flow fib mismatch");

  // add or delte rx flow on interface.
  if (dpi_add_del_rx_flow (hw_if_index, rx_flow_id, is_add, is_ipv6))
    return clib_error_return (0, "error %s flow",
			      is_add ? "enabling" : "disabling");

  // enable or disable offload mode.
  // enable/disable ip[4|6]-unicast arc's dpi[4|6]-flow-input node.
  dpi_flow_offload_mode (hw_if_index, is_ipv6, is_enable);

  return 0;
}
```

**set ipv4 flow bypass**
```
VLIB_CLI_COMMAND (dpi_set_ip4_flow_bypass_command, static) =
{
  .path = "dpi set ip4 flow-bypass",
  .short_help = "dpi set ip4 flow-bypass <interface> [del]",
  .function = dpi_set_ip4_flow_bypass_command_fn,
};

static clib_error_t *
dpi_set_ip4_flow_bypass_command_fn (vlib_main_t * vm,
				    unformat_input_t * input,
				    vlib_cli_command_t * cmd)
{
  return dpi_set_flow_bypass (0, input, cmd);
}

static clib_error_t *
dpi_set_flow_bypass (u32 is_ip6,
		     unformat_input_t * input, vlib_cli_command_t * cmd)
{
  unformat_input_t _line_input, *line_input = &_line_input;
  vnet_main_t *vnm = vnet_get_main ();
  clib_error_t *error = 0;
  u32 sw_if_index, is_enable;

  sw_if_index = ~0;
  is_enable = 1;

  if (!unformat_user (input, unformat_line_input, line_input))
    return 0;

  while (unformat_check_input (line_input) != UNFORMAT_END_OF_INPUT)
  {
	  // parse interface.
      if (unformat_user (line_input, unformat_vnet_sw_interface, vnm,
			 &sw_if_index));
	  // add or delete.
      else if (unformat (line_input, "del"))
		is_enable = 0;
	  // error occurred.
      else
	  {
	    error = unformat_parse_error (line_input);
	    goto done;
	  }
  }

  // check interface.
  if (~0 == sw_if_index)
  {
      error = clib_error_return (0, "unknown interface `%U'",
				 format_unformat_error, line_input);
      goto done;
  }

  // enable or disable bypass mode. 
  // enable/disable ip-unicast arc's dpi[4|6]-input node.
  dpi_flow_bypass_mode (sw_if_index, is_ip6, is_enable);

done:
  unformat_free (line_input);

  return error;
}
```

**set ipv6 flow bypass**
```
VLIB_CLI_COMMAND (dpi_set_ip6_flow_bypass_command, static) =
{
    .path = "dpi set ip6 flow-bypass",
    .short_help = "dpi set ip6 flow-bypass <interface> [del]",
    .function = dpi_set_ip6_flow_bypass_command_fn,
};

static clib_error_t *
dpi_set_ip6_flow_bypass_command_fn (vlib_main_t * vm,
				    unformat_input_t * input,
				    vlib_cli_command_t * cmd)
{
  return dpi_set_flow_bypass (0, input, cmd);
}
```

### 处理

#### 节点

```
ip4-unicast/ip6-unicast arcs
            |
            |
dpi4-input/dpi4-flow-input/dpi6-input/dpi6-flow-input     
            |
            |
ip4-lookup/ip6-lookup
```

#### plugins/udpi/dpi_node.c
```
VNET_FEATURE_INIT (dpi4_input, static) =
{
  .arc_name = "ip4-unicast",
  .node_name = "dpi4-input",
  .runs_before = VNET_FEATURES ("ip4-lookup"),
};

VNET_FEATURE_INIT (dpi6_input, static) =
{
  .arc_name = "ip6-unicast",
  .node_name = "dpi6-input",
  .runs_before = VNET_FEATURES ("ip6-lookup"),
};

VNET_FEATURE_INIT (dpi4_flow_input, static) =
{
  .arc_name = "ip4-unicast",
  .node_name = "dpi4-flow-input",
  .runs_before = VNET_FEATURES ("ip4-lookup"),
};

VNET_FEATURE_INIT (dpi6_flow_input, static) =
{
  .arc_name = "ip6-unicast",
  .node_name = "dpi6-flow-input",
  .runs_before = VNET_FEATURES ("ip6-lookup"),
};
```

**dpi4-input**
```
VLIB_REGISTER_NODE (dpi4_input_node) =
{
  .name = "dpi4-input",
  .vector_size = sizeof (u32),
  .n_errors = DPI_INPUT_N_ERROR,
  .error_strings = dpi_input_error_strings,
  .n_next_nodes = DPI_INPUT_N_NEXT,
  .next_nodes = {
#define _(s,n) [DPI_INPUT_NEXT_##s] = n,
    foreach_dpi_input_next
#undef _
  },
  .format_trace = format_dpi_rx_trace,
};

VLIB_NODE_FN (dpi4_input_node) (vlib_main_t * vm,
                      vlib_node_runtime_t * node,
                      vlib_frame_t * frame)
{
  return dpi_input_inline (vm, node, frame, /* is_ip4 */ 1);
}

always_inline uword
dpi_input_inline (vlib_main_t * vm,
                  vlib_node_runtime_t * node,
                  vlib_frame_t * frame, u32 is_ip4)
{
  dpi_main_t *dm = &dpi_main;
  u32 *from, *to_next, n_left_from, n_left_to_next, next_index;

  from = vlib_frame_vector_args (frame);
  n_left_from = frame->n_vectors;
  next_index = node->cached_next_index;

  while (n_left_from > 0)
    {
      vlib_get_next_frame (vm, node, next_index, to_next, n_left_to_next);

      while (n_left_from > 0 && n_left_to_next > 0)
    {
      u32 bi0, next0 = 0;
      vlib_buffer_t *b0;
      ip4_header_t *ip40;
      ip6_header_t *ip60;
      tcp_header_t *tcp0;
      udp_header_t *udp0;
      dpi4_flow_key_t key40;
      dpi6_flow_key_t key60;
      u32 fib_index0 = ~0;
      u64 flow_id0 = ~0;
      u32 flow_index0 = ~0;
      int not_found0 = 0;
      u8 is_reverse0 = 0;
      dpi_flow_entry_t *flow0 = 0;
      u32 ip_len0, l4_len0, payload_len0;
      u8 protocol0;
      u8 *l4_pkt0, *payload0;
      u16 dst_port = 0;
      segment *seg = 0;
      segment *prev_seg = 0;

      bi0 = to_next[0] = from[0];
      b0 = vlib_get_buffer (vm, bi0);
      ip_len0 = vlib_buffer_length_in_chain (vm, b0);

      if (is_ip4)
        {
          ip40 = vlib_buffer_get_current (b0);
          ip4_main_t *im4 = &ip4_main;
          // find fib index by rx interface.
          fib_index0 = vec_elt (im4->fib_index_by_sw_if_index,
                                vnet_buffer(b0)->sw_if_index[VLIB_RX]);
          // parse packet and lookup flow(create new flow when not found), return flow id.
          parse_ip4_packet_and_lookup(ip40, fib_index0, &key40,
                                      &not_found0, &flow_id0);
        }
      else
        {
          ip60 = vlib_buffer_get_current (b0);
          ip6_main_t *im6 = &ip6_main;
          // find fib index by rx interface.
          fib_index0 = vec_elt (im6->fib_index_by_sw_if_index,
                                vnet_buffer(b0)->sw_if_index[VLIB_RX]);
          // parse packet and lookup flow(create new flow when not found), return flow id.
          parse_ip6_packet_and_lookup(ip60, fib_index0, &key60,
                                      &not_found0, &flow_id0);
        }

      // the flow id's highest bit is 1 for the reverse flow.
      is_reverse0 = (u8)((flow_id0 >> 63) & 0x1);
      // get flow entry by flow id.
      flow_index0 = (u32)(flow_id0 & (u32)(~0));
      flow0 = pool_elt_at_index (dm->dpi_flows, flow_index0);

      /* have detected successfully, directly return */
      if(flow0->info->detect_done)
          goto enqueue0;

      /* check layer4 */
      if (is_ip4)
      {
          l4_pkt0 = (u8 *)(ip40 + 1);
          l4_len0 = ip_len0 - sizeof(ip4_header_t);
          protocol0 = ip40->protocol;
      }
      else
      {
          l4_pkt0 = (u8 *)(ip60 + 1);
          l4_len0 = ip_len0 - sizeof(ip6_header_t);
          protocol0 = ip60->protocol;
      }

      if((protocol0 == IP_PROTOCOL_TCP) && (l4_len0 >= 20))
      {
          tcp0 = (tcp_header_t *)l4_pkt0;
          payload_len0 = l4_len0 - tcp_doff(tcp0) * 4;
          payload0 = l4_pkt0 + tcp_doff(tcp0) * 4;
          dst_port = tcp0->dst_port;
      }
      else if ((protocol0 == IP_PROTOCOL_UDP) && (l4_len0 >= 8))
      {
          udp0 = (udp_header_t *)l4_pkt0;
          payload_len0 = l4_len0 - sizeof(udp_header_t);
          payload0 = l4_pkt0 + sizeof(udp_header_t);
          dst_port = udp0->dst_port;
      }
      else
      {
          payload_len0 = l4_len0;
          payload0 = l4_pkt0;
      }

      // flow's layer4 infos.
      flow0->info->l4_protocol = protocol0;
      flow0->info->dst_port = dst_port;

      /* TCP stream reassembly and detect a protocol pdu */
      if((protocol0 == IP_PROTOCOL_TCP) && (flow0->reass_en))
      {
          // do tcp reassembly and detect layer 7 application. 
          // dpi_handle_tcp_stream()->dpi_handle_tcp_segments()->dpi_detect_application()
          dpi_handle_tcp_stream(flow0, bi0, l4_pkt0, payload_len0, is_reverse0);

          /* This packet has been consumed, retrieve next packet */
          if(flow0->consumed)
            goto trace0;

          /* send out continuous scanned segments */
          seg = flow0->first_seg;
          // add tcp segments to the next node queue.
          dpi_enqueue_tcp_segments(seg,vm,node,next_index,to_next,n_left_to_next,bi0,next0);
          flow0->first_seg = 0;

          /* Here detected successfully, send out remaining segments in seg_queue */
          if(flow0->info->detect_done)
          {
              // client to server segments queue.
              seg = flow0->c2s.seg_queue;
              // add tcp segments to the next node queue.
              dpi_enqueue_tcp_segments(seg,vm,node,next_index,to_next,n_left_to_next,bi0,next0);
              flow0->c2s.seg_queue = 0;

              // server to client segments queue.
              seg = flow0->s2c.seg_queue;
              // add tcp segments to the next node queue.
              dpi_enqueue_tcp_segments(seg,vm,node,next_index,to_next,n_left_to_next,bi0,next0);
              flow0->s2c.seg_queue = 0;
          }
          goto trace0;
      }
      else
      {
          // to detect directly except for tcp.
          /* detect layer 7 application for single packet */
          dpi_detect_application (payload0, payload_len0, flow0->info);
      }

enqueue0:
      to_next[0] = bi0;
      to_next++;
      n_left_to_next--;
      next0 = flow0->next_index;
      vlib_validate_buffer_enqueue_x1 (vm, node, next_index,
                       to_next, n_left_to_next,
                       bi0, next0);

trace0:
      if (PREDICT_FALSE (b0->flags & VLIB_BUFFER_IS_TRACED))
      {
          dpi_rx_trace_t *tr
            = vlib_add_trace (vm, node, b0, sizeof (*tr));
          tr->app_id = flow0->info->app_id;
          tr->next_index = next0;
          tr->error = b0->error;
          tr->flow_id = flow_index0;
      }

      from += 1;
      n_left_from -= 1;
    }

    vlib_put_next_frame (vm, node, next_index, n_left_to_next);
  }

  return frame->n_vectors;
}

static inline int
parse_ip4_packet_and_lookup (ip4_header_t * ip4, u32 fib_index,
                             dpi4_flow_key_t * key4,
                             int * not_found, u64 * flow_id)
{
  dpi_main_t *dm = &dpi_main;
  u8 protocol = ip4_is_fragment (ip4) ? 0xfe : ip4->protocol;
  u16 src_port = 0;
  u16 dst_port = 0;
  dpi_flow_entry_t *flow;

  // uint64 key[0] composed by src_address and dst_address.
  key4->key[0] = ip4->src_address.as_u32
      | (((u64) ip4->dst_address.as_u32) << 32);

  if (protocol == IP_PROTOCOL_UDP || protocol == IP_PROTOCOL_TCP)
    {
      /* tcp and udp ports have the same offset */
      udp_header_t * udp = ip4_next_header (ip4);
      src_port = udp->src_port;
      dst_port = udp->dst_port;
    }

  // uint64 key[1] composed by protocol, src_port and dst_port.
  key4->key[1] = (((u64) protocol) << 32) | ((u32) src_port << 16) | dst_port;
  // uint64 key[2] composed by fib_index.
  key4->key[2] = (u64) fib_index;

  // init value of the key[3]->value pair.
  key4->value = ~0;
  // find value by key4 from dpi flows. return find or not.
  *not_found = clib_bihash_search_inline_24_8 (&dm->dpi4_flow_by_key, key4);
  // value is flow_id which found by key4.
  *flow_id = key4->value;

  /* not found, then create new SW flow dynamically */
  if (*not_found)
  {
      int add_failed;
      // allocate a dpi flow entry.
      pool_get_aligned(dm->dpi_flows, flow, CLIB_CACHE_LINE_BYTES);
      // init the dpi flow entry.
      clib_memset(flow, 0, sizeof(*flow));
      // set flow id by current flow's offset.
      *flow_id = flow - dm->dpi_flows;

      // set next node index is ip4-lookup.
      flow->next_index = DPI_INPUT_NEXT_IP4_LOOKUP;
      // init flow index.
      flow->flow_index = ~0;

      // allocate a flow info.
      pool_get_aligned(dm->dpi_infos, flow->info, CLIB_CACHE_LINE_BYTES);
      // init flow info.
      clib_memset(flow->info, 0, sizeof(*flow->info));
      // init flow info's app_id.
      flow->info->app_id = ~0;

      /* Add forwarding flow entry */
      key4->value = *flow_id;
      // add flow entry to bihash, and return ok or failed.
      add_failed = clib_bihash_add_del_24_8 (&dm->dpi4_flow_by_key, key4,
                                             1 /*add */);
      // add failed.
      if (add_failed)
        {
          // free flow info and flow.
          pool_put(dm->dpi_infos, flow->info);
          pool_put(dm->dpi_flows, flow);
          return -1;
        }

      /* Add reverse flow entry*/
      key4->key[0] = ip4->dst_address.as_u32
          | (((u64) ip4->src_address.as_u32) << 32);
      key4->key[1] = (((u64) protocol) << 32) | ((u32) dst_port << 16)
          | src_port;
      key4->key[2] = (u64) fib_index;
      // !!!notes: the highest bit of the value(flow id) is 1 for the reverse flow.
      key4->value = (u64) flow_id | ((u64) 1 << 63);
      add_failed = clib_bihash_add_del_24_8 (&dm->dpi4_flow_by_key, key4,
                                             1 /*add */);

      if (add_failed)
        {
          pool_put(dm->dpi_infos, flow->info);
          pool_put(dm->dpi_flows, flow);
          return -1;
        }

      /* Open a Hyperscan stream for each flow */
      hs_error_t err = hs_open_stream (dm->default_db.database, 0,
                                       &(flow->info->stream));

      if (err != HS_SUCCESS)
        {
          // free flow info and flow.
          pool_put(dm->dpi_infos, flow->info);
          pool_put(dm->dpi_flows, flow);
          return -1;
        }
  }

  return 0;
}
```

```
void
dpi_detect_application (u8 *payload, u32 payload_len,
                        dpi_flow_info_t *flow)
{

  /* detect if payload is SSL's payload for default port */
  dpi_search_tcp_ssl(payload, payload_len, flow);

  /* TBD: add detect if is SSL's payload with non default port*/

}
```
**dpi4-flow-input**

ip4-flow-input处理流程和ip4-input处理流程相同，区别在于ip4-flow-input依赖硬件网卡的flow director特性将flow处理卸载（offload）到网卡中，提高性能。

```
VLIB_REGISTER_NODE (dpi4_flow_input_node) = {
  .name = "dpi4-flow-input",
  .type = VLIB_NODE_TYPE_INTERNAL,
  .vector_size = sizeof (u32),

  .format_trace = format_dpi_rx_trace,

  .n_errors = DPI_FLOW_N_ERROR,
  .error_strings = dpi_flow_error_strings,

  .n_next_nodes = DPI_FLOW_N_NEXT,
  .next_nodes = {
#define _(s,n) [DPI_FLOW_NEXT_##s] = n,
    foreach_dpi_flow_input_next
#undef _
  },
};

VLIB_NODE_FN (dpi4_flow_input_node) (vlib_main_t * vm,
                     vlib_node_runtime_t * node,
                     vlib_frame_t * frame)
{
  return dpi_flow_input_inline (vm, node, frame, /* is_ip4 */ 1);
}

always_inline uword
dpi_flow_input_inline (vlib_main_t * vm,
               vlib_node_runtime_t * node,
               vlib_frame_t * frame, u32 is_ip4)
{
  dpi_main_t *dm = &dpi_main;
  u32 *from, *to_next, n_left_from, n_left_to_next, next_index;

  from = vlib_frame_vector_args (frame);
  n_left_from = frame->n_vectors;
  next_index = node->cached_next_index;

  while (n_left_from > 0)
    {
      vlib_get_next_frame (vm, node, next_index, to_next, n_left_to_next);

      while (n_left_from > 0 && n_left_to_next > 0)
    {
      u32 bi0, next0 = 0;
      vlib_buffer_t *b0;
      ip4_header_t *ip40;
      ip6_header_t *ip60;
      tcp_header_t *tcp0;
      udp_header_t *udp0;
      u32 flow_id0 = ~0;
      u32 flow_index0 = ~0;
      u32 is_reverse0 = 0;
      dpi_flow_entry_t *flow0;
      u32 ip_len0, l4_len0, payload_len0;
      u8 protocol0;
      u8 *l4_pkt0, *payload0;
      u16 dst_port = 0;
      segment *seg = 0;
      segment *prev_seg = 0;

      bi0 = to_next[0] = from[0];
      b0 = vlib_get_buffer (vm, bi0);
      vlib_buffer_advance (b0, +(sizeof (ethernet_header_t)));
      ip_len0 = vlib_buffer_length_in_chain (vm, b0);

      if (is_ip4)
        {
          ip40 = vlib_buffer_get_current (b0);
          dpi_check_ip4 (ip40, ip_len0);
        }
      else
        {
          ip60 = vlib_buffer_get_current (b0);
          dpi_check_ip6 (ip60, ip_len0);
        }

      flow_id0 = b0->flow_id - dm->flow_id_start;

      is_reverse0 = (u32) ((flow_id0 >> 31) & 0x1);
      flow_index0 = (u32) (flow_id0 & (u32) (~(1 << 31)));
      flow0 = pool_elt_at_index (dm->dpi_flows, flow_index0);

      /* have detected successfully, directly return */
      if (flow0->info->detect_done)
        goto enqueue0;

      /* check layer4 */
      if (is_ip4)
        {
          l4_pkt0 = (u8 *) (ip40 + 1);
          l4_len0 = ip_len0 - sizeof (ip4_header_t);
          protocol0 = ip40->protocol;
        }
      else
        {
          l4_pkt0 = (u8 *) (ip60 + 1);
          l4_len0 = ip_len0 - sizeof (ip6_header_t);
          protocol0 = ip60->protocol;
        }

      if ((protocol0 == IP_PROTOCOL_TCP) && (l4_len0 >= 20))
        {
          tcp0 = (tcp_header_t *) l4_pkt0;
          payload_len0 = l4_len0 - tcp_doff (tcp0) * 4;
          payload0 = l4_pkt0 + tcp_doff (tcp0) * 4;
          dst_port = tcp0->dst_port;
        }
      else if ((protocol0 == IP_PROTOCOL_UDP) && (l4_len0 >= 8))
        {
          udp0 = (udp_header_t *) l4_pkt0;
          payload_len0 = l4_len0 - sizeof (udp_header_t);
          payload0 = l4_pkt0 + sizeof (udp_header_t);
          dst_port = udp0->dst_port;
        }
      else
        {
          payload_len0 = l4_len0;
          payload0 = l4_pkt0;
        }

      flow0->info->l4_protocol = protocol0;
      flow0->info->dst_port = dst_port;

      /* TCP stream reassembly and detect a protocol pdu */
      if ((protocol0 == IP_PROTOCOL_TCP) && (flow0->reass_en))
        {
          dpi_handle_tcp_stream (flow0, bi0, l4_pkt0, payload_len0,
                     is_reverse0);

          /* This packet has been consumed, retrieve next packet */
          if (flow0->consumed)
        goto trace0;

          /* send out continuous scanned segments */
          seg = flow0->first_seg;
          dpi_enqueue_tcp_segments (seg, vm, node, next_index, to_next,
                    n_left_to_next, bi0, next0);
          flow0->first_seg = 0;

          /* Here detected successfully, send out remaining segments in seg_queue */
          if (flow0->info->detect_done)
        {
          seg = flow0->c2s.seg_queue;
          dpi_enqueue_tcp_segments (seg, vm, node, next_index,
                        to_next, n_left_to_next, bi0,
                        next0);
          flow0->c2s.seg_queue = 0;

          seg = flow0->s2c.seg_queue;
          dpi_enqueue_tcp_segments (seg, vm, node, next_index,
                        to_next, n_left_to_next, bi0,
                        next0);
          flow0->s2c.seg_queue = 0;
        }
          goto trace0;
        }
      else
        {
          /* detect layer 7 application for single packet */
          dpi_detect_application (payload0, payload_len0, flow0->info);
        }

enqueue0:
      to_next[0] = bi0;
      to_next++;
      n_left_to_next--;
      next0 = flow0->next_index;
      vlib_validate_buffer_enqueue_x1 (vm, node, next_index,
                       to_next, n_left_to_next,
                       bi0, next0);

trace0:
      if (PREDICT_FALSE (b0->flags & VLIB_BUFFER_IS_TRACED))
        {
          dpi_rx_trace_t *tr
        = vlib_add_trace (vm, node, b0, sizeof (*tr));
          tr->app_id = flow0->info->app_id;
          tr->next_index = next0;
          tr->error = b0->error;
          tr->flow_id = flow_index0;
        }

      from += 1;
      n_left_from -= 1;
    }

      vlib_put_next_frame (vm, node, next_index, n_left_to_next);
    }

  return frame->n_vectors;
}
```
**dpi6-input**

流程同dpi4-input。

**dpi6-flow-input**

流程同dpi6-flow-input。
