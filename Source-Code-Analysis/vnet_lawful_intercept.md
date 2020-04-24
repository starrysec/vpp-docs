## 合法监听

### 配置
源代码位于`vnet/lawful-intercept/lawful-intercept.c`文件。

```
VLIB_CLI_COMMAND (set_li_command, static) = {
    .path = "set li",
    .short_help =
    "set li src <ip4-address> collector <ip4-address> udp-port <nnnn>",
    .function = set_li_command_fn,
};
```

```
static clib_error_t *
set_li_command_fn (vlib_main_t * vm,
		   unformat_input_t * input, vlib_cli_command_t * cmd)
{
  li_main_t *lm = &li_main;
  ip4_address_t collector;
  u8 collector_set = 0;
  ip4_address_t src;
  u8 src_set = 0;
  u32 tmp;
  u16 udp_port = 0;
  u8 is_add = 1;
  int i;

  while (unformat_check_input (input) != UNFORMAT_END_OF_INPUT)
  {
    // 解析collector ip地址
    if (unformat (input, "collector %U", unformat_ip4_address, &collector))
      collector_set = 1;
    // 设置源ip地址
    if (unformat (input, "src %U", unformat_ip4_address, &src))
      src_set = 1;
    // 解析collector udp端口
    else if (unformat (input, "udp-port %d", &tmp))
      udp_port = tmp;
    // 新建or删除
    else if (unformat (input, "del"))
      is_add = 0;
    else
      break;
  }

  // collector ip，port和src ip必须设置
  if (collector_set == 0)
    return clib_error_return (0, "collector must be set...");
  if (src_set == 0)
    return clib_error_return (0, "src must be set...");
  if (udp_port == 0)
    return clib_error_return (0, "udp-port must be set...");

  // 新增
  if (is_add == 1)
  {
    for (i = 0; i < vec_len (lm->collectors); i++)
	{
      // collector ip地址或者端口已经配置了
	  if (lm->collectors[i].as_u32 == collector.as_u32)
	  {
	    if (lm->ports[i] == udp_port)
	      return clib_error_return (0, "collector %U:%d already configured", &collector, udp_port);
	    else
	      return clib_error_return (0, "collector %U already configured with port %d", &collector, (int) (lm->ports[i]));
	  }
	}
    // 添加collector ip和端口，src ip到vector
    vec_add1 (lm->collectors, collector);
    vec_add1 (lm->ports, udp_port);
    vec_add1 (lm->src_addrs, src);
    return 0;
  }
  // 删除
  else
  {
    // 从vector中删除collector
    for (i = 0; i < vec_len (lm->collectors); i++)
	{
	  if ((lm->collectors[i].as_u32 == collector.as_u32)
	      && lm->ports[i] == udp_port)
	    {
	      vec_delete (lm->collectors, 1, i);
	      vec_delete (lm->ports, 1, i);
	      vec_delete (lm->src_addrs, 1, i);
	      return 0;
	    }
	}
    return clib_error_return (0, "collector %U:%d not configured", &collector, udp_port);
  }
  return 0;
}
```

### 处理