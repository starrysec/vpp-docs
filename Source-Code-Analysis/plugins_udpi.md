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



