## ipfix-export

### 配置

#### vnet/ipfix-export/flow_report.c

配置exporter，使能flow reporting process。
```
VLIB_CLI_COMMAND (set_ipfix_exporter_command, static) = {
    .path = "set ipfix exporter",
    .short_help = "set ipfix exporter "
                  "collector <ip4-address> [port <port>] "
                  "src <ip4-address> [fib-id <fib-id>] "
                  "[path-mtu <path-mtu>] "
                  "[template-interval <template-interval>]",
                  "[udp-checksum]",
    .function = set_ipfix_exporter_command_fn,
};

static clib_error_t *
set_ipfix_exporter_command_fn (vlib_main_t * vm,
			       unformat_input_t * input,
			       vlib_cli_command_t * cmd)
{
  flow_report_main_t *frm = &flow_report_main;
  ip4_address_t collector, src;
  u16 collector_port = UDP_DST_PORT_ipfix;
  u32 fib_id;
  u32 fib_index = ~0;

  collector.as_u32 = 0;
  src.as_u32 = 0;
  u32 path_mtu = 512;		// RFC 7011 section 10.3.3.
  u32 template_interval = 20;
  u8 udp_checksum = 0;

  while (unformat_check_input (input) != UNFORMAT_END_OF_INPUT)
  {
	  // 解析collector ip
      if (unformat (input, "collector %U", unformat_ip4_address, &collector));
	  // 解析collector port
      else if (unformat (input, "port %u", &collector_port));
	  // 解析ipfix src ip
      else if (unformat (input, "src %U", unformat_ip4_address, &src));
	  // 解析fib id
      else if (unformat (input, "fib-id %u", &fib_id))
	  {
		ip4_main_t *im = &ip4_main;
		uword *p = hash_get (im->fib_index_by_table_id, fib_id);
		if (!p)
			return clib_error_return (0, "fib ID %d doesn't exist\n", fib_id);
		fib_index = p[0];
	  }
	  // 解析最大mtu
	  else if (unformat (input, "path-mtu %u", &path_mtu));
	  // 解析模板更新间隔
      else if (unformat (input, "template-interval %u", &template_interval));
	  // 解析校验和标志
      else if (unformat (input, "udp-checksum"))
		udp_checksum = 1;
      else
		break;
  }

  if (collector.as_u32 != 0 && src.as_u32 == 0)
    return clib_error_return (0, "src address required");

  // vpp ipfix不支持分片，mtu最大1450
  if (path_mtu > 1450 /* vpp does not support fragmentation */ )
    return clib_error_return (0, "too big path-mtu value, maximum is 1450");

  // mtu最小68
  if (path_mtu < 68)
    return clib_error_return (0, "too small path-mtu value, minimum is 68");

  /* Reset report streams if we are reconfiguring IP addresses */
  if (frm->ipfix_collector.as_u32 != collector.as_u32 ||
      frm->src_address.as_u32 != src.as_u32 ||
      frm->collector_port != collector_port)
	vnet_flow_reports_reset (frm);

  // 赋值
  frm->ipfix_collector.as_u32 = collector.as_u32;
  frm->collector_port = collector_port;
  frm->src_address.as_u32 = src.as_u32;
  frm->fib_index = fib_index;
  frm->path_mtu = path_mtu;
  frm->template_interval = template_interval;
  frm->udp_checksum = udp_checksum;

  if (collector.as_u32)
    vlib_cli_output (vm, "Collector %U, src address %U, "
		     "fib index %d, path MTU %u, "
		     "template resend interval %us, "
		     "udp checksum %s",
		     format_ip4_address, &frm->ipfix_collector,
		     format_ip4_address, &frm->src_address,
		     fib_index, path_mtu, template_interval,
		     udp_checksum ? "enabled" : "disabled");
  else
    vlib_cli_output (vm, "IPFIX Collector is disabled");

  /* Turn on the flow reporting process */
  // 使能flow reporting处理
  vlib_process_signal_event (vm, flow_report_process_node.index, 1, 0);
  return 0;
}
```

