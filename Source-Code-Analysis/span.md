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

  /* 新增或者禁用镜像条目 */
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

#### vnet/span/node.c

```
VLIB_REGISTER_NODE (span_input_node) = {
  span_node_defs,
  .name = "span-input",
};

VLIB_REGISTER_NODE (span_output_node) = {
  span_node_defs,
  .name = "span-output",
};

VLIB_REGISTER_NODE (span_l2_input_node) = {
  span_node_defs,
  .name = "span-l2-input",
};

VLIB_REGISTER_NODE (span_l2_output_node) = {
  span_node_defs,
  .name = "span-l2-output",
};
```

```
static_always_inline uword
span_node_inline_fn (vlib_main_t * vm, vlib_node_runtime_t * node,
		     vlib_frame_t * frame, vlib_rx_or_tx_t rxtx,
		     span_feat_t sf)
{
  span_main_t *sm = &span_main;
  vnet_main_t *vnm = vnet_get_main ();
  u32 n_left_from, *from, *to_next;
  u32 next_index;
  u32 sw_if_index;
  static __thread vlib_frame_t **mirror_frames = 0;

  from = vlib_frame_vector_args (frame);
  n_left_from = frame->n_vectors;
  next_index = node->cached_next_index;

  vec_validate_aligned (mirror_frames, sm->max_sw_if_index,
			CLIB_CACHE_LINE_BYTES);

  while (n_left_from > 0)
    {
      u32 n_left_to_next;

      vlib_get_next_frame (vm, node, next_index, to_next, n_left_to_next);

      while (n_left_from >= 4 && n_left_to_next >= 2)
	{
	  u32 bi0;
	  u32 bi1;
	  vlib_buffer_t *b0;
	  vlib_buffer_t *b1;
	  u32 sw_if_index0;
	  u32 next0 = 0;
	  u32 sw_if_index1;
	  u32 next1 = 0;

	  /* speculatively enqueue b0, b1 to the current next frame */
	  to_next[0] = bi0 = from[0];
	  to_next[1] = bi1 = from[1];
	  to_next += 2;
	  n_left_to_next -= 2;
	  from += 2;
	  n_left_from -= 2;

	  b0 = vlib_get_buffer (vm, bi0);
	  b1 = vlib_get_buffer (vm, bi1);
	  /* 获取数据包b0在rx后者tx方向的接口索引 */
	  sw_if_index0 = vnet_buffer (b0)->sw_if_index[rxtx];
	  sw_if_index1 = vnet_buffer (b1)->sw_if_index[rxtx];

	  /* 构造镜像后的镜像frames */
	  span_mirror (vm, node, sw_if_index0, b0, mirror_frames, rxtx, sf);
	  span_mirror (vm, node, sw_if_index1, b1, mirror_frames, rxtx, sf);

	  switch (sf)
	    {
		/* 从L2镜像 */
	    case SPAN_FEAT_L2:
	      if (rxtx == VLIB_RX)
		{
		  /* 根据配置获取next节点 */
		  next0 = vnet_l2_feature_next (b0, sm->l2_input_next,
						L2INPUT_FEAT_SPAN);
		  next1 = vnet_l2_feature_next (b1, sm->l2_input_next,
						L2INPUT_FEAT_SPAN);
		}
	      else
		{
		  /* 根据配置获取next节点 */
		  next0 = vnet_l2_feature_next (b0, sm->l2_output_next,
						L2OUTPUT_FEAT_SPAN);
		  next1 = vnet_l2_feature_next (b1, sm->l2_output_next,
						L2OUTPUT_FEAT_SPAN);
		}
	      break;
		/* 从device镜像 */
	    case SPAN_FEAT_DEVICE:
	    default:
		  /* 根据配置获取next节点 */
	      vnet_feature_next (&next0, b0);
	      vnet_feature_next (&next1, b1);
	      break;
	    }
		
      /* 验证处理结果 */
	  /* verify speculative enqueue, maybe switch current next frame */
	  vlib_validate_buffer_enqueue_x2 (vm, node, next_index,
					   to_next, n_left_to_next,
					   bi0, bi1, next0, next1);
	}
      while (n_left_from > 0 && n_left_to_next > 0)
	{
	  u32 bi0;
	  vlib_buffer_t *b0;
	  u32 sw_if_index0;
	  u32 next0 = 0;

	  /* speculatively enqueue b0 to the current next frame */
	  to_next[0] = bi0 = from[0];
	  to_next += 1;
	  n_left_to_next -= 1;
	  from += 1;
	  n_left_from -= 1;

	  b0 = vlib_get_buffer (vm, bi0);
	  sw_if_index0 = vnet_buffer (b0)->sw_if_index[rxtx];

	  span_mirror (vm, node, sw_if_index0, b0, mirror_frames, rxtx, sf);

	  switch (sf)
	    {
	    case SPAN_FEAT_L2:
	      if (rxtx == VLIB_RX)
		next0 = vnet_l2_feature_next (b0, sm->l2_input_next,
					      L2INPUT_FEAT_SPAN);
	      else
		next0 = vnet_l2_feature_next (b0, sm->l2_output_next,
					      L2OUTPUT_FEAT_SPAN);
	      break;
	    case SPAN_FEAT_DEVICE:
	    default:
	      vnet_feature_next (&next0, b0);
	      break;
	    }

	  /* verify speculative enqueue, maybe switch current next frame */
	  vlib_validate_buffer_enqueue_x1 (vm, node, next_index, to_next,
					   n_left_to_next, bi0, next0);
	}
	  /* 所有流程都正确处理完毕后，下一结点的frame上已经有本结点处理过后的数据索引
         执行该函数，将相关信息登记到vlib_pending_frame_t中，准备开始调度处理 
	  */
      vlib_put_next_frame (vm, node, next_index, n_left_to_next);
    }

  /* 按照配置，输出镜像frames到next节点处理 */
  for (sw_if_index = 0; sw_if_index < vec_len (mirror_frames); sw_if_index++)
    {
	  /* 根据监控端口，后去镜像frame */
      vlib_frame_t *f = mirror_frames[sw_if_index];
      if (f == 0)
		continue;
      
	  /* 从L2镜像 */
      if (sf == SPAN_FEAT_L2)
		/* frame发往下一node，处理后输出 */
		vlib_put_frame_to_node (vm, l2output_node.index, f);
	  /* 从device镜像 */
      else
	    /* frame发往监端口，进行输出 */
		vnet_put_frame_to_sw_interface (vnm, sw_if_index, f);
	  /* 重置镜像frames */
	  mirror_frames[sw_if_index] = 0;
    }
	
  /* 返回frame中数据包个数 */
  return frame->n_vectors;
}
```