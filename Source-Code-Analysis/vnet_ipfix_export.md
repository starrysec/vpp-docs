## ipfix-export

### 配置

#### vnet/ipfix-export/flow_report.c

配置exporter，使能flow reporting处理流程
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
	  // 解析是否计算校验和标志
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
  // 使能flow reporting处理流程
  vlib_process_signal_event (vm, flow_report_process_node.index, 1, 0);
  return 0;
}
```

### 刷新

#### vnet/ipfix-export/flow_report.c

```
VLIB_CLI_COMMAND (ipfix_flush_command, static) = {
    .path = "ipfix flush",
    .short_help = "flush the current ipfix data [for make test]",
    .function = ipfix_flush_command_fn,
};

static clib_error_t *
ipfix_flush_command_fn (vlib_main_t * vm,
			unformat_input_t * input, vlib_cli_command_t * cmd)
{
  /* poke the flow reporting process */
  // 通知flow reporting处理流程
  vlib_process_signal_event (vm, flow_report_process_node.index, 1, 0);
  return 0;
}
```

### 处理

#### vnet/ipfix-export/flow_report.c

```
VLIB_REGISTER_NODE (flow_report_process_node) = {
    .function = flow_report_process,
    .type = VLIB_NODE_TYPE_PROCESS,
    .name = "flow-report-process",
};

static uword
flow_report_process (vlib_main_t * vm,
		     vlib_node_runtime_t * rt, vlib_frame_t * f)
{
  flow_report_main_t *frm = &flow_report_main;
  flow_report_t *fr;
  u32 ip4_lookup_node_index;
  vlib_node_t *ip4_lookup_node;
  vlib_frame_t *nf = 0;
  u32 template_bi;
  u32 *to_next;
  int send_template;
  f64 now;
  int rv;
  uword event_type;
  uword *event_data = 0;

  /* Wait for Godot... */
  // 等待事件或者超时
  vlib_process_wait_for_event_or_clock (vm, 1e9);
  event_type = vlib_process_get_events (vm, &event_data);
  if (event_type != 1)
    clib_warning ("bogus kickoff event received, %d", event_type);
  vec_reset_length (event_data);

  /* Enqueue pkts to ip4-lookup */
  // 获取ip-lookup的node索引
  ip4_lookup_node = vlib_get_node_by_name (vm, (u8 *) "ip4-lookup");
  ip4_lookup_node_index = ip4_lookup_node->index;

  while (1)
  {
	  // 等待事件或者超时
      vlib_process_wait_for_event_or_clock (vm, 5.0);
      event_type = vlib_process_get_events (vm, &event_data);
      vec_reset_length (event_data);

      // 遍历reports
      vec_foreach (fr, frm->reports)
      {
		now = vlib_time_now (vm);

		/* Need to send a template packet? */
		send_template = now > (fr->last_template_sent + frm->template_interval);
		send_template += fr->last_template_sent == 0;
		// template数据包索引
		template_bi = ~0;
		rv = 0;

		// 时间间隔到了，构造template数据包
		if (send_template)
		    // 返回template数据包索引
			rv = send_template_packet (frm, fr, &template_bi);

        // 构造template失败
		if (rv < 0)
			continue;

		// 获取传递给ip4-lookup node的frame
		nf = vlib_get_frame_to_node (vm, ip4_lookup_node_index);
		nf->n_vectors = 0;
		// 获取frame vector的数据指针
		to_next = vlib_frame_vector_args (nf);

		if (template_bi != ~0)
		{
		    // tempate数据包索引加入frame vector中
			to_next[0] = template_bi;
			to_next++;
			// frame vector数据包个数加1
			nf->n_vectors++;
		}
        
        // 回调其他模块注册的数据刷新插件，用于构造data set数据。
        // 可用的data callback有：
        // 1.flowprobe中的flowprobe_data_callback_ip4，flowprobe_data_callback_ip6，flowprobe_data_callback_l2
        // 2.ipfix-export中的ipfix_classify_send_flows
		nf = fr->flow_data_callback (frm, fr, nf, to_next, ip4_lookup_node_index);
		// 把frame传给下个节点：ip4-lookup，最后发送出去
		if (nf)
			vlib_put_frame_to_node (vm, ip4_lookup_node_index, nf);
	  }
  }

  return 0;			/* not so much */
}

int
send_template_packet (flow_report_main_t * frm,
		      flow_report_t * fr, u32 * buffer_indexp)
{
  u32 bi0;
  // 数据包缓冲区
  vlib_buffer_t *b0;
  // template结构
  ip4_ipfix_template_packet_t *tp;
  // ipfix头
  ipfix_message_header_t *h;
  ip4_header_t *ip;
  udp_header_t *udp;
  vlib_main_t *vm = frm->vlib_main;
  flow_report_stream_t *stream;

  ASSERT (buffer_indexp);

  // 初始状态必须是要rewrite且rewrite字符串不为空，如果不是则进行状态修正
  if (fr->update_rewrite || fr->rewrite == 0)
  {
    // 缺少collector ip或者src ip，则disable掉flow-report-process节点
    if (frm->ipfix_collector.as_u32 == 0 || frm->src_address.as_u32 == 0)
	{
	  vlib_node_set_state (frm->vlib_main, flow_report_process_node.index,
			       VLIB_NODE_STATE_DISABLED);
	  return -1;
	}
	// 修正为初始状态
    vec_free (fr->rewrite);
    fr->update_rewrite = 1;
  }

  // 要rewrite
  if (fr->update_rewrite)
  {
    // 调用其他模块注册的rewrite回调函数：
    // 可用的rewrite_callback有：
    // 1.flowprobe中的flowprobe_template_rewrite_l2，flowprobe_template_rewrite_ip4，flowprobe_template_rewrite_ip6
    // 2.ipfix-export中的ipfix_classify_template_rewrite
    // 这里的rewrite字符串为ipfix中的template data，即除了ipfix头16字节外的数据。
	fr->rewrite = fr->rewrite_callback (frm, fr,
					  &frm->ipfix_collector,
					  &frm->src_address,
					  frm->collector_port,
					  fr->report_elements,
					  fr->n_report_elements,
					  fr->stream_indexp);
	// 已rewrite过了，不需要rewrite了
    fr->update_rewrite = 0;
  }

  // 申请数据包内存，返回数据包索引
  if (vlib_buffer_alloc (vm, &bi0, 1) != 1)
    return -1;
  // 获取数据包地址
  b0 = vlib_get_buffer (vm, bi0);

  /* Initialize the buffer */
  VLIB_BUFFER_TRACE_TRAJECTORY_INIT (b0);

  ASSERT (vec_len (fr->rewrite) < vlib_buffer_get_default_data_size (vm));

  // 将teplate data追加到ipfix头16字节之后。
  clib_memcpy_fast (b0->data, fr->rewrite, vec_len (fr->rewrite));
  b0->current_data = 0;
  b0->current_length = vec_len (fr->rewrite);
  b0->flags |= (VLIB_BUFFER_TOTAL_LENGTH_VALID | VNET_BUFFER_F_FLOW_REPORT);
  vnet_buffer (b0)->sw_if_index[VLIB_RX] = 0;
  vnet_buffer (b0)->sw_if_index[VLIB_TX] = frm->fib_index;

  tp = vlib_buffer_get_current (b0);
  ip = (ip4_header_t *) & tp->ip4;
  udp = (udp_header_t *) (ip + 1);
  // 获取ipfix头，填充16字节
  h = (ipfix_message_header_t *) (udp + 1);

  /* FIXUP: message header export_time */
  h->export_time = (u32)
    (((f64) frm->unix_time_0) +
     (vlib_time_now (frm->vlib_main) - frm->vlib_time_0));
  h->export_time = clib_host_to_net_u32 (h->export_time);

  stream = &frm->streams[fr->stream_index];

  /* FIXUP: message header sequence_number. Templates do not increase it */
  h->sequence_number = clib_host_to_net_u32 (stream->sequence_number);

  /* FIXUP: udp length */
  udp->length = clib_host_to_net_u16 (b0->current_length - sizeof (*ip));

  // 计算udp校验和
  if (frm->udp_checksum)
  {
    /* RFC 7011 section 10.3.2. */
    udp->checksum = ip4_tcp_udp_compute_checksum (vm, b0, ip);
    if (udp->checksum == 0)
		udp->checksum = 0xffff;
  }

  // 返回数据包索引
  *buffer_indexp = bi0;

  // 更新上次template sent时间
  fr->last_template_sent = vlib_time_now (vm);

  return 0;
}
```
