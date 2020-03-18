## span

### 配置

#### vnet/span/span.c

```
VLIB_CLI_COMMAND (set_interface_span_command, static) = {
  .path = "set interface span",
  .short_help = "set interface span <if-name> [l2] {disable | destination <if-name> [both|rx|tx]}",
  .function = set_interface_span_command_fn,
};
```

```
static clib_error_t *
set_interface_span_command_fn (vlib_main_t * vm,
			       unformat_input_t * input,
			       vlib_cli_command_t * cmd)
{
  span_main_t *sm = &span_main;
  /* 源端口，即镜像端口 */
  u32 src_sw_if_index = ~0;
  /* 目的端口，即监控端口 */
  u32 dst_sw_if_index = ~0;
  /* 镜像方式，从device还是从L2 */
  span_feat_t sf = SPAN_FEAT_DEVICE;
  /* 镜像状态，rx|tx|both|disable */
  span_state_t state = SPAN_BOTH;
  /* 状态位，保证只设置一次镜像状态 */
  int state_set = 0;

  while (unformat_check_input (input) != UNFORMAT_END_OF_INPUT)
    {
	  /* 解析镜像端口 */
      if (unformat (input, "%U", unformat_vnet_sw_interface,
		    sm->vnet_main, &src_sw_if_index))
	;
	  /* 解析监控端口 */
      else if (unformat (input, "destination %U", unformat_vnet_sw_interface,
			 sm->vnet_main, &dst_sw_if_index))
	;
	  /* 解析镜像状态(rx,tx,both或者disable) */
      else if (unformat (input, "%U", unformat_span_state, &state))
	{
	  /* 置状态位，保证只设置一次镜像状态 */
	  if (state_set)
	    return clib_error_return (0, "Multiple mirror states in input");
	  state_set = 1;
	}
	  /* 解析镜像方式，默认从device，可以指定从L2 */
      else if (unformat (input, "l2"))
	sf = SPAN_FEAT_L2;
	  /* 错误 */
      else
	return clib_error_return (0, "Invalid input");
    }

  /* 新增或者删除镜像条目 */
  int rv =
    span_add_delete_entry (vm, src_sw_if_index, dst_sw_if_index, state, sf);
  if (rv == VNET_API_ERROR_INVALID_INTERFACE)
    return clib_error_return (0, "Invalid interface");
  return 0;
}
```

```
int
span_add_delete_entry (vlib_main_t * vm,
		       u32 src_sw_if_index, u32 dst_sw_if_index, u8 state,
		       span_feat_t sf)
{
  span_main_t *sm = &span_main;

  if (state > SPAN_BOTH)
    return VNET_API_ERROR_UNIMPLEMENTED;

  if ((src_sw_if_index == ~0) || (dst_sw_if_index == ~0 && state > 0)
      || (src_sw_if_index == dst_sw_if_index))
    return VNET_API_ERROR_INVALID_INTERFACE;

  vec_validate_aligned (sm->interfaces, src_sw_if_index,
			CLIB_CACHE_LINE_BYTES);

  /* 根据镜像接口索引，查找镜像配置结构体 */
  span_interface_t *si = vec_elt_at_index (sm->interfaces, src_sw_if_index);

  /* 获取rx和tx状态 */
  int rx = ! !(state & SPAN_RX);
  int tx = ! !(state & SPAN_TX);

  /* 获取镜像配置结构体中，rx和tx方向的镜像配置结构 */
  span_mirror_t *rxm = &si->mirror_rxtx[sf][VLIB_RX];
  span_mirror_t *txm = &si->mirror_rxtx[sf][VLIB_TX];

  /* 设置监控端口，并获取上次rx方向监控端口数量 */
  u32 last_rx_ports_count = span_dst_set (rxm, dst_sw_if_index, rx);
  /* 设置监控端口，并获取上次tx方向监控端口数量 */
  u32 last_tx_ports_count = span_dst_set (txm, dst_sw_if_index, tx);

  /* 同一个镜像端口，上次rx方向监控端口数量为0，现在为1，则使能rx镜像功能 */
  int enable_rx = last_rx_ports_count == 0 && rxm->num_mirror_ports == 1;
  /* 同一个镜像端口，上次rx方向监控端口数量大于0，现在为0，则禁用rx镜像功能 */
  int disable_rx = last_rx_ports_count > 0 && rxm->num_mirror_ports == 0;
  /* 同一个镜像端口，上次tx方向监控端口数量为0，现在为1，则使能tx镜像功能 */
  int enable_tx = last_tx_ports_count == 0 && txm->num_mirror_ports == 1;
  /* 同一个镜像端口，上次tx方向监控端口数量大于0，现在为0，则禁用tx镜像功能 */
  int disable_tx = last_tx_ports_count > 0 && txm->num_mirror_ports == 0;

  switch (sf)
    {
	/* 从device镜像 */
    case SPAN_FEAT_DEVICE:
      if (enable_rx || disable_rx)
	        /* 启用或者禁用rx镜像 */
			vnet_feature_enable_disable ("device-input", "span-input",
				     src_sw_if_index, rx, 0, 0);
      if (enable_tx || disable_tx)
			/* 启用或者禁用tx镜像 */
			vnet_feature_enable_disable ("interface-output", "span-output",
				     src_sw_if_index, tx, 0, 0);
      break;
	/* 从L2镜像 */
    case SPAN_FEAT_L2:
      if (enable_rx || disable_rx)
	        /* 启用或者禁用rx镜像 */
			l2input_intf_bitmap_enable (src_sw_if_index, L2INPUT_FEAT_SPAN, rx);
      if (enable_tx || disable_tx)
			/* 启用或者禁用tx镜像 */
			l2output_intf_bitmap_enable (src_sw_if_index, L2OUTPUT_FEAT_SPAN, tx);
      break;
	/* 错误 */
    default:
      return VNET_API_ERROR_UNIMPLEMENTED;
    }

  if (dst_sw_if_index != ~0 && dst_sw_if_index > sm->max_sw_if_index)
    sm->max_sw_if_index = dst_sw_if_index;

  return 0;
}
```

```
VLIB_CLI_COMMAND (show_interfaces_span_command, static) = {
  .path = "show interface span",
  .short_help = "Shows SPAN mirror table",
  .function = show_interfaces_span_command_fn,
};
```

### 处理
