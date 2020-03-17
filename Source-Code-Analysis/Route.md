## 路由(Route)

### 添加路由

`vnet/ip/lookup.c`

```
VLIB_CLI_COMMAND (ip_route_command, static) = {
  .path = "ip route",
  .short_help = "ip route [add|del] [count <n>] <dst-ip-addr>/<width> [table <table-id>] via [next-hop-address] [next-hop-interface] [next-hop-table <value>] [weight <value>] [preference <value>] [udp-encap-id <value>] [ip4-lookup-in-table <value>] [ip6-lookup-in-table <value>] [mpls-lookup-in-table <value>] [resolve-via-host] [resolve-via-connected] [rx-ip4 <interface>] [out-labels <value value value>]",
  .function = vnet_ip_route_cmd,
  .is_mp_safe = 1,
};
```

```
static clib_error_t *
vnet_ip_route_cmd (vlib_main_t * vm,
		   unformat_input_t * main_input, vlib_cli_command_t * cmd)
{
  unformat_input_t _line_input, *line_input = &_line_input;
  u32 table_id, is_del, fib_index, payload_proto;
  dpo_id_t dpo = DPO_INVALID, *dpos = NULL;
  fib_route_path_t *rpaths = NULL, rpath;
  fib_prefix_t *prefixs = NULL, pfx;
  clib_error_t *error = NULL;
  f64 count;
  int i;

  is_del = 0;
  table_id = 0;
  count = 1;
  clib_memset (&pfx, 0, sizeof (pfx));

  /* Get a line of input. */
  if (!unformat_user (main_input, unformat_line_input, line_input))
    return 0;

  while (unformat_check_input (line_input) != UNFORMAT_END_OF_INPUT)
    {
      clib_memset (&rpath, 0, sizeof (rpath));
      /* 解析路由表ID */
      if (unformat (line_input, "table %d", &table_id))
	;
      else if (unformat (line_input, "count %f", &count))
	;
      /* 解析目标网络（IPv4）前缀和前缀长度 */
      else if (unformat (line_input, "%U/%d",
			 unformat_ip4_address, &pfx.fp_addr.ip4, &pfx.fp_len))
	{
	  payload_proto = pfx.fp_proto = FIB_PROTOCOL_IP4;
	  vec_add1 (prefixs, pfx);
	}
      /* 解析目标网络（IPv6）前缀和前缀长度 */
      else if (unformat (line_input, "%U/%d",
			 unformat_ip6_address, &pfx.fp_addr.ip6, &pfx.fp_len))
	{
	  payload_proto = pfx.fp_proto = FIB_PROTOCOL_IP6;
	  vec_add1 (prefixs, pfx);
	}
      /* 解析下一跳 */
      else if (unformat (line_input, "via %U",
			 unformat_fib_route_path, &rpath, &payload_proto))
	{
	  vec_add1 (rpaths, rpath);
	}
      /* 有前缀，则根据下一跳创建dpo */
      else if (vec_len (prefixs) > 0 &&
	       unformat (line_input, "via %U",
			 unformat_dpo, &dpo, prefixs[0].fp_proto))
	{
	  vec_add1 (dpos, dpo);
	}
      /* 添加路由 */
      else if (unformat (line_input, "del"))
	is_del = 1;
      /* 删除路由 */ 
      else if (unformat (line_input, "add"))
	is_del = 0;
      /* 动作错误 */
      else
	{
	  error = unformat_parse_error (line_input);
	  goto done;
	}
    }
    
  /* 前缀错误 */
  if (vec_len (prefixs) == 0)
    {
      error =
	clib_error_return (0, "expected ip4/ip6 destination address/length.");
      goto done;
    }
  
  /* 检查：添加路由时， 下一跳和dpo至少有一个存在 */
  if (!is_del && vec_len (rpaths) + vec_len (dpos) == 0)
    {
      error = clib_error_return (0, "expected paths.");
      goto done;
    }

  /* 未明确指定table id的话，使用默认值：0 */
  if (~0 == table_id)
    {
      /*
       * if no table_id is passed we will manipulate the default
       */
      fib_index = 0;
    }
  else
    {
      /* 根据协议和table id查找路由表是否存在 */
      fib_index = fib_table_find (prefixs[0].fp_proto, table_id);

      /* 路由表不存在 */
      if (~0 == fib_index)
	{
	  error = clib_error_return (0, "Nonexistent table id %d", table_id);
	  goto done;
	}
    }

  /* 遍历前缀，添加路由到路由表或者从路由表中删除路由 */
  for (i = 0; i < vec_len (prefixs); i++)
    {
      /* 删除路由，只需要前缀即可 */
      if (is_del && 0 == vec_len (rpaths))
	{
      /* 从路由表中删除路由 */
	  fib_table_entry_delete (fib_index, &prefixs[i], FIB_SOURCE_CLI);
	}
      /* 添加路由 */
      else if (!is_del && 1 == vec_len (dpos))
	{
      /* 根据前缀和dpo添加路由 */
	  fib_table_entry_special_dpo_add (fib_index,
					   &prefixs[i],
					   FIB_SOURCE_CLI,
					   FIB_ENTRY_FLAG_EXCLUSIVE,
					   &dpos[0]);
	  /* 重置dpo */
      dpo_reset (&dpos[0]);
	}
      /* 错误：多个dpo表示下一跳要负载均衡，不支持 */
      else if (vec_len (dpos) > 0)
	{
	  error =
	    clib_error_return (0,
			       "Load-balancing over multiple special adjacencies is unsupported");
	  goto done;
	}
      /*  */
      else if (0 < vec_len (rpaths))
	{
	  u32 k, n, incr;
	  ip46_address_t dst = prefixs[i].fp_addr;
	  f64 t[2];
	  n = count;
	  t[0] = vlib_time_now (vm);
	  incr = 1 << ((FIB_PROTOCOL_IP4 == prefixs[0].fp_proto ? 32 : 128) -
		       prefixs[i].fp_len);

	  for (k = 0; k < n; k++)
	    {
	      fib_prefix_t rpfx = {
		.fp_len = prefixs[i].fp_len,
		.fp_proto = prefixs[i].fp_proto,
		.fp_addr = dst,
	      };

	      if (is_del)
		fib_table_entry_path_remove2 (fib_index,
					      &rpfx, FIB_SOURCE_CLI, rpaths);
	      else
		fib_table_entry_path_add2 (fib_index,
					   &rpfx,
					   FIB_SOURCE_CLI,
					   FIB_ENTRY_FLAG_NONE, rpaths);

	      if (FIB_PROTOCOL_IP4 == prefixs[0].fp_proto)
		{
		  dst.ip4.as_u32 =
		    clib_host_to_net_u32 (incr +
					  clib_net_to_host_u32 (dst.
								ip4.as_u32));
		}
	      else
		{
		  int bucket = (incr < 64 ? 0 : 1);
		  dst.ip6.as_u64[bucket] =
		    clib_host_to_net_u64 (incr +
					  clib_net_to_host_u64 (dst.ip6.as_u64
								[bucket]));
		}
	    }

	  t[1] = vlib_time_now (vm);
	  if (count > 1)
	    vlib_cli_output (vm, "%.6e routes/sec", count / (t[1] - t[0]));
	}
      /* 错误 */
      else
	{
	  error = clib_error_return (0, "Don't understand what you want...");
	  goto done;
	}
    }

    /* 释放资源 */
done:
  vec_free (dpos);
  vec_free (prefixs);
  vec_free (rpaths);
  unformat_free (line_input);
  return error;
}
```

### 删除路由

### 路由查找
