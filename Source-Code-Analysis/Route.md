## 路由(Route)

### 添加/删除路由

### vnet/ip/lookup.c

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
      /* count含义未知，暂时无法分析  */
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

```
uword
unformat_fib_route_path (unformat_input_t * input, va_list * args)
{
    fib_route_path_t *rpath = va_arg (*args, fib_route_path_t *);
    dpo_proto_t *payload_proto = va_arg (*args, void*);
    u32 weight, preference, udp_encap_id, fi;
    mpls_label_t out_label;
    vnet_main_t *vnm;

    vnm = vnet_get_main ();
    clib_memset(rpath, 0, sizeof(*rpath));
    rpath->frp_weight = 1;
    rpath->frp_sw_if_index = ~0;

    while (unformat_check_input (input) != UNFORMAT_END_OF_INPUT)
    {
        if (unformat (input, "%U %U",
                      unformat_ip4_address,
                      &rpath->frp_addr.ip4,
                      unformat_vnet_sw_interface, vnm,
                      &rpath->frp_sw_if_index))
        {
            rpath->frp_proto = DPO_PROTO_IP4;
        }
        else if (unformat (input, "%U %U",
                           unformat_ip6_address,
                           &rpath->frp_addr.ip6,
                           unformat_vnet_sw_interface, vnm,
                           &rpath->frp_sw_if_index))
        {
            rpath->frp_proto = DPO_PROTO_IP6;
        }
        else if (unformat (input, "weight %u", &weight))
        {
            rpath->frp_weight = weight;
        }
        else if (unformat (input, "preference %u", &preference))
        {
            rpath->frp_preference = preference;
        }
        else if (unformat (input, "%U next-hop-table %d",
                           unformat_ip4_address,
                           &rpath->frp_addr.ip4,
                           &rpath->frp_fib_index))
        {
            rpath->frp_sw_if_index = ~0;
            rpath->frp_proto = DPO_PROTO_IP4;

            /*
             * the user enter table-ids, convert to index
             */
            fi = fib_table_find (FIB_PROTOCOL_IP4, rpath->frp_fib_index);
            if (~0 == fi)
                return 0;
            rpath->frp_fib_index = fi;
        }
        else if (unformat (input, "%U next-hop-table %d",
                           unformat_ip6_address,
                           &rpath->frp_addr.ip6,
                           &rpath->frp_fib_index))
        {
            rpath->frp_sw_if_index = ~0;
            rpath->frp_proto = DPO_PROTO_IP6;
            fi = fib_table_find (FIB_PROTOCOL_IP6, rpath->frp_fib_index);
            if (~0 == fi)
                return 0;
            rpath->frp_fib_index = fi;
        }
        else if (unformat (input, "%U",
                           unformat_ip4_address,
                           &rpath->frp_addr.ip4))
        {
            /*
             * the recursive next-hops are by default in the default table
             */
            rpath->frp_fib_index = 0;
            rpath->frp_sw_if_index = ~0;
            rpath->frp_proto = DPO_PROTO_IP4;
        }
        else if (unformat (input, "%U",
                           unformat_ip6_address,
                           &rpath->frp_addr.ip6))
        {
            rpath->frp_fib_index = 0;
            rpath->frp_sw_if_index = ~0;
            rpath->frp_proto = DPO_PROTO_IP6;
        }
        else if (unformat (input, "udp-encap %d", &udp_encap_id))
        {
            rpath->frp_udp_encap_id = udp_encap_id;
            rpath->frp_flags |= FIB_ROUTE_PATH_UDP_ENCAP;
            rpath->frp_proto = *payload_proto;
        }
        else if (unformat (input, "lookup in table %d", &rpath->frp_fib_index))
        {
            rpath->frp_proto = *payload_proto;
            rpath->frp_sw_if_index = ~0;
            rpath->frp_flags |= FIB_ROUTE_PATH_DEAG;
        }
        else if (unformat (input, "resolve-via-host"))
        {
            rpath->frp_flags |= FIB_ROUTE_PATH_RESOLVE_VIA_HOST;
        }
        else if (unformat (input, "resolve-via-attached"))
        {
            rpath->frp_flags |= FIB_ROUTE_PATH_RESOLVE_VIA_ATTACHED;
        }
        else if (unformat (input, "pop-pw-cw"))
        {
            rpath->frp_flags |= FIB_ROUTE_PATH_POP_PW_CW;
        }
        else if (unformat (input,
                           "ip4-lookup-in-table %d",
                           &rpath->frp_fib_index))
        {
            rpath->frp_proto = DPO_PROTO_IP4;
            *payload_proto = DPO_PROTO_IP4;
            fi = fib_table_find (FIB_PROTOCOL_IP4, rpath->frp_fib_index);
            if (~0 == fi)
                return 0;
            rpath->frp_fib_index = fi;
        }
        else if (unformat (input,
                           "ip6-lookup-in-table %d",
                           &rpath->frp_fib_index))
        {
            rpath->frp_proto = DPO_PROTO_IP6;
            *payload_proto = DPO_PROTO_IP6;
            fi = fib_table_find (FIB_PROTOCOL_IP6, rpath->frp_fib_index);
            if (~0 == fi)
                return 0;
            rpath->frp_fib_index = fi;
        }
        else if (unformat (input,
                           "mpls-lookup-in-table %d",
                           &rpath->frp_fib_index))
        {
            rpath->frp_proto = DPO_PROTO_MPLS;
            *payload_proto = DPO_PROTO_MPLS;
            fi = fib_table_find (FIB_PROTOCOL_MPLS, rpath->frp_fib_index);
            if (~0 == fi)
                return 0;
            rpath->frp_fib_index = fi;
        }
        else if (unformat (input, "src-lookup"))
        {
            rpath->frp_flags |= FIB_ROUTE_PATH_SOURCE_LOOKUP;
        }
        else if (unformat (input,
                           "l2-input-on %U",
                           unformat_vnet_sw_interface, vnm,
                           &rpath->frp_sw_if_index))
        {
            rpath->frp_proto = DPO_PROTO_ETHERNET;
            *payload_proto = DPO_PROTO_ETHERNET;
            rpath->frp_flags |= FIB_ROUTE_PATH_INTF_RX;
        }
        else if (unformat (input, "via-label %U",
                           unformat_mpls_unicast_label,
                           &rpath->frp_local_label))
        {
            rpath->frp_eos = MPLS_NON_EOS;
            rpath->frp_proto = DPO_PROTO_MPLS;
            rpath->frp_sw_if_index = ~0;
        }
        else if (unformat (input, "rx-ip4 %U",
                           unformat_vnet_sw_interface, vnm,
                           &rpath->frp_sw_if_index))
        {
            rpath->frp_proto = DPO_PROTO_IP4;
            rpath->frp_flags = FIB_ROUTE_PATH_INTF_RX;
        }
      else if (unformat (input, "local"))
	{
	  clib_memset (&rpath->frp_addr, 0, sizeof (rpath->frp_addr));
	  rpath->frp_sw_if_index = ~0;
	  rpath->frp_weight = 1;
	  rpath->frp_flags |= FIB_ROUTE_PATH_LOCAL;
        }
      else if (unformat (input, "%U",
			 unformat_mfib_itf_flags, &rpath->frp_mitf_flags))
	;
      else if (unformat (input, "out-labels"))
        {
            while (unformat (input, "%U",
                             unformat_mpls_unicast_label, &out_label))
            {
                fib_mpls_label_t fml = {
                    .fml_value = out_label,
                };
                vec_add1(rpath->frp_label_stack, fml);
            }
        }
        else if (unformat (input, "%U",
                           unformat_vnet_sw_interface, vnm,
                           &rpath->frp_sw_if_index))
        {
            rpath->frp_proto = *payload_proto;
        }
        else if (unformat (input, "via"))
        {
            /* new path, back up and return */
            unformat_put_input (input);
            unformat_put_input (input);
            unformat_put_input (input);
            unformat_put_input (input);
            break;
        }
        else
        {
            return (0);
        }
    }

    return (1);
}
```

```
static uword
unformat_dpo (unformat_input_t * input, va_list * args)
{
  /* 要返回的dpo */
  dpo_id_t *dpo = va_arg (*args, dpo_id_t *);
  /* 路由表协议，ipv4或者ipv6或者mpls */
  fib_protocol_t fp = va_arg (*args, int);
  /* dpo协议 */
  dpo_proto_t proto;

  /* 路由表协议转成dpo协议，ip4->ip4, ip6->ip6, mpls->mpls */
  proto = fib_proto_to_dpo (fp);

  if (unformat (input, "drop"))
    dpo_copy (dpo, drop_dpo_get (proto));
  else if (unformat (input, "punt"))
    dpo_copy (dpo, punt_dpo_get (proto));
  else if (unformat (input, "local"))
    receive_dpo_add_or_lock (proto, ~0, NULL, dpo);
  else if (unformat (input, "null-send-unreach"))
    ip_null_dpo_add_and_lock (proto, IP_NULL_ACTION_SEND_ICMP_UNREACH, dpo);
  else if (unformat (input, "null-send-prohibit"))
    ip_null_dpo_add_and_lock (proto, IP_NULL_ACTION_SEND_ICMP_PROHIBIT, dpo);
  else if (unformat (input, "null"))
    ip_null_dpo_add_and_lock (proto, IP_NULL_ACTION_NONE, dpo);
  else if (unformat (input, "classify"))
    {
      u32 classify_table_index;

      if (!unformat (input, "%d", &classify_table_index))
	{
	  clib_warning ("classify adj must specify table index");
	  return 0;
	}

      dpo_set (dpo, DPO_CLASSIFY, proto,
	       classify_dpo_create (proto, classify_table_index));
    }
  else
    return 0;

  return 1;
}
```

### vnet/fib/fib_table.c

```
fib_node_index_t
fib_table_entry_special_dpo_add (u32 fib_index,
                                 const fib_prefix_t *prefix,
                                 fib_source_t source,
                                 fib_entry_flag_t flags,
                                 const dpo_id_t *dpo)
{
    /* 路由表项id */
    fib_node_index_t fib_entry_index;
    /* 路由表 */
    fib_table_t *fib_table;

    /* 通过路由表id和协议获取路由表 */
    fib_table = fib_table_get(fib_index, prefix->fp_proto);
    /* 在路由表中根据前缀查找路由表项 */
    fib_entry_index = fib_table_lookup_exact_match_i(fib_table, prefix);

    /* 路由表项不存在，则添加 */
    if (FIB_NODE_INDEX_INVALID == fib_entry_index)
    {
    /* 创建路由表项 */
	fib_entry_index = fib_entry_create_special(fib_index, prefix,
						   source, flags,
						   dpo);

    /* 在路由表中插入路由表项 */
	fib_table_entry_insert(fib_table, prefix, fib_entry_index);
        /* 路由源类型加一 */
        fib_table_source_count_inc(fib_table, source);
    }
    /* 路由表项存在 */
    else
    {
        int was_sourced;

        /* 查找是否有source这种路由源 */
        was_sourced = fib_entry_is_sourced(fib_entry_index, source);
        /* 添加source这样路由源的路由表项到路由表 */
	fib_entry_special_add(fib_entry_index, source, flags, dpo);

        /* 添加后，再次确认是否有source这种路由源 */
        if (was_sourced != fib_entry_is_sourced(fib_entry_index, source))
        {
        /* 添加好了，source这种路由源类型加一 */
        fib_table_source_count_inc(fib_table, source);
        }
    }

    /* 返回路由表项id */
    return (fib_entry_index);
}
```

### 路由查找
