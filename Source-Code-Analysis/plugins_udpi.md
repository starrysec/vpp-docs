## UDPI

udpi是基于vpp的深度数据包检测插件。

### 配置

#### plugins/udpi/dpi_cli.c

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

### 处理

