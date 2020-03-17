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
    /* 路由路径 */
    fib_route_path_t *rpath = va_arg (*args, fib_route_path_t *);
    /* dpo协议 */
    dpo_proto_t *payload_proto = va_arg (*args, void*);
    /* weight: 权重，preference: ?, udp_encap_id: ?, fi: ? */
    u32 weight, preference, udp_encap_id, fi;
    /* mpls外层标签 */
    mpls_label_t out_label;
    vnet_main_t *vnm;

    vnm = vnet_get_main ();
    /* 初始化rpath */
    clib_memset(rpath, 0, sizeof(*rpath));
    /* 给rpath权重和出接口赋初值 */
    rpath->frp_weight = 1;
    rpath->frp_sw_if_index = ~0;

    while (unformat_check_input (input) != UNFORMAT_END_OF_INPUT)
    {
        /* 解析ipv4下一跳地址[next-hop-address]和出接口[next-hop-interface] */
        if (unformat (input, "%U %U",
                      unformat_ip4_address,
                      &rpath->frp_addr.ip4,
                      unformat_vnet_sw_interface, vnm,
                      &rpath->frp_sw_if_index))
        {
            rpath->frp_proto = DPO_PROTO_IP4;
        }
        /* 解析ipv6下一跳地址[next-hop-address]和出接口[next-hop-interface] */ 
        else if (unformat (input, "%U %U",
                           unformat_ip6_address,
                           &rpath->frp_addr.ip6,
                           unformat_vnet_sw_interface, vnm,
                           &rpath->frp_sw_if_index))
        {
            rpath->frp_proto = DPO_PROTO_IP6;
        }
        /* 解析权重 */
        else if (unformat (input, "weight %u", &weight))
        {
            rpath->frp_weight = weight;
        }
        else if (unformat (input, "preference %u", &preference))
        {
            rpath->frp_preference = preference;
        }
        /* 解析ipv4递归路由出接口，和下一跳[next-hop-table]*/
        else if (unformat (input, "%U next-hop-table %d",
                           unformat_ip4_address,
                           &rpath->frp_addr.ip4,
                           &rpath->frp_fib_index))
        {
            /* 出接口 */
            rpath->frp_sw_if_index = ~0;
            /* dpo协议 */
            rpath->frp_proto = DPO_PROTO_IP4;

			/* 用户可见的table id转为内部index */
            /*
             * the user enter table-ids, convert to index
             */
            fi = fib_table_find (FIB_PROTOCOL_IP4, rpath->frp_fib_index);
            if (~0 == fi)
                return 0;
			/* rpath所属路由表index */
            rpath->frp_fib_index = fi;
        }
		/* 解析ipv4递归路由出接口，和下一跳[next-hop-table]*/
        else if (unformat (input, "%U next-hop-table %d",
                           unformat_ip6_address,
                           &rpath->frp_addr.ip6,
                           &rpath->frp_fib_index))
        {
			/* 出接口 */
            rpath->frp_sw_if_index = ~0;
			/* dpo协议 */
            rpath->frp_proto = DPO_PROTO_IP6;
			/* 用户可见的table id转为内部index */
            fi = fib_table_find (FIB_PROTOCOL_IP6, rpath->frp_fib_index);
            if (~0 == fi)
                return 0;
			/* rpath所属路由表index */
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
		/* 指定ipv4查找路由表id */
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
		/* 指定ipv6查找路由表id */
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
		/* 指定mpls查找路由表id */
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

#### vnet/ip/ip4_forward.c

```
/* *INDENT-OFF* */
VLIB_REGISTER_NODE (ip4_lookup_node) =
{
  .name = "ip4-lookup",
  .vector_size = sizeof (u32),
  .format_trace = format_ip4_lookup_trace,
  .n_next_nodes = IP_LOOKUP_N_NEXT,
  .next_nodes = IP4_LOOKUP_NEXT_NODES,
};
/* *INDENT-ON* */
```

```
VLIB_NODE_FN (ip4_lookup_node) (vlib_main_t * vm, vlib_node_runtime_t * node,
				vlib_frame_t * frame)
{
  return ip4_lookup_inline (vm, node, frame);
}
```

```
always_inline uword
ip4_lookup_inline (vlib_main_t * vm,
		   vlib_node_runtime_t * node, vlib_frame_t * frame)
{
  ip4_main_t *im = &ip4_main;
  vlib_combined_counter_main_t *cm = &load_balance_main.lbm_to_counters;
  u32 n_left, *from;
  u32 thread_index = vm->thread_index;
  /* 数据包缓冲区数组 */
  vlib_buffer_t *bufs[VLIB_FRAME_SIZE];
  /* 数据包缓存指针 */
  vlib_buffer_t **b = bufs;
  /* 存储处理每个数据包的下个node，首地址next
  u16 nexts[VLIB_FRAME_SIZE], *next;

  /* frame起始地址 */
  from = vlib_frame_vector_args (frame);
  /* frame中数据包个数 */
  n_left = frame->n_vectors;
  next = nexts;
  /* 将frame中的所有数据包buffer索引转换为指针 */
  vlib_get_buffers (vm, from, bufs, n_left);

/* 缓存行填充的数据包个数大于等于8时(默认值16)*/
#if (CLIB_N_PREFETCHES >= 8)
  /* frame中数据包个数大于等于4个时 */
  while (n_left >= 4)
    {
	  /* 第1到4个数据包的ip头 */
      ip4_header_t *ip0, *ip1, *ip2, *ip3;
	  /* 第1到4个数据包相关的lb结构,lb是一组dpo组成，每个dpo代表了数据包的一个走向*/
      const load_balance_t *lb0, *lb1, *lb2, *lb3;
	  /* 第1到4个数据包的转发结构(8-8-8-8 mtrie树) */
      ip4_fib_mtrie_t *mtrie0, *mtrie1, *mtrie2, *mtrie3;
	  /* 第1到4个数据包的转发结构(8-8-8-8 mtrie树)叶子节点 */
      ip4_fib_mtrie_leaf_t leaf0, leaf1, leaf2, leaf3;
	  /* 第1到4个数据包的dst地址 */
      ip4_address_t *dst_addr0, *dst_addr1, *dst_addr2, *dst_addr3;
	  /* 第1到4个数据包相关的lb结构索引 */
      u32 lb_index0, lb_index1, lb_index2, lb_index3;
	  /* 第1到4个数据包flow hash配置 */
      flow_hash_config_t flow_hash_config0, flow_hash_config1;
      flow_hash_config_t flow_hash_config2, flow_hash_config3;
	  /* 第1到4个数据包的流hash值 */
      u32 hash_c0, hash_c1, hash_c2, hash_c3;
	  /* 第1到4个数据包相关的dpo结构 */
      const dpo_id_t *dpo0, *dpo1, *dpo2, *dpo3;

	  /* 预取下次迭代使用的后4个数据包 */
      /* Prefetch next iteration. */
      if (n_left >= 8)
	{
	  vlib_prefetch_buffer_header (b[4], LOAD);
	  vlib_prefetch_buffer_header (b[5], LOAD);
	  vlib_prefetch_buffer_header (b[6], LOAD);
	  vlib_prefetch_buffer_header (b[7], LOAD);

	  CLIB_PREFETCH (b[4]->data, sizeof (ip0[0]), LOAD);
	  CLIB_PREFETCH (b[5]->data, sizeof (ip0[0]), LOAD);
	  CLIB_PREFETCH (b[6]->data, sizeof (ip0[0]), LOAD);
	  CLIB_PREFETCH (b[7]->data, sizeof (ip0[0]), LOAD);
	}

	  /* 获取第1到4个数据包的ip头 */
      ip0 = vlib_buffer_get_current (b[0]);
      ip1 = vlib_buffer_get_current (b[1]);
      ip2 = vlib_buffer_get_current (b[2]);
      ip3 = vlib_buffer_get_current (b[3]);

	  /* 获取第1到4个数据包的dst地址 */
      dst_addr0 = &ip0->dst_address;
      dst_addr1 = &ip1->dst_address;
      dst_addr2 = &ip2->dst_address;
      dst_addr3 = &ip3->dst_address;

	  /* 设置数据包缓冲区的路由表索引 */
      ip_lookup_set_buffer_fib_index (im->fib_index_by_sw_if_index, b[0]);
      ip_lookup_set_buffer_fib_index (im->fib_index_by_sw_if_index, b[1]);
      ip_lookup_set_buffer_fib_index (im->fib_index_by_sw_if_index, b[2]);
      ip_lookup_set_buffer_fib_index (im->fib_index_by_sw_if_index, b[3]);

	  /* 获取第1到4个数据包转发表结构 */
      mtrie0 = &ip4_fib_get (vnet_buffer (b[0])->ip.fib_index)->mtrie;
      mtrie1 = &ip4_fib_get (vnet_buffer (b[1])->ip.fib_index)->mtrie;
      mtrie2 = &ip4_fib_get (vnet_buffer (b[2])->ip.fib_index)->mtrie;
      mtrie3 = &ip4_fib_get (vnet_buffer (b[3])->ip.fib_index)->mtrie;

      leaf0 = ip4_fib_mtrie_lookup_step_one (mtrie0, dst_addr0);
      leaf1 = ip4_fib_mtrie_lookup_step_one (mtrie1, dst_addr1);
      leaf2 = ip4_fib_mtrie_lookup_step_one (mtrie2, dst_addr2);
      leaf3 = ip4_fib_mtrie_lookup_step_one (mtrie3, dst_addr3);

      leaf0 = ip4_fib_mtrie_lookup_step (mtrie0, leaf0, dst_addr0, 2);
      leaf1 = ip4_fib_mtrie_lookup_step (mtrie1, leaf1, dst_addr1, 2);
      leaf2 = ip4_fib_mtrie_lookup_step (mtrie2, leaf2, dst_addr2, 2);
      leaf3 = ip4_fib_mtrie_lookup_step (mtrie3, leaf3, dst_addr3, 2);

      leaf0 = ip4_fib_mtrie_lookup_step (mtrie0, leaf0, dst_addr0, 3);
      leaf1 = ip4_fib_mtrie_lookup_step (mtrie1, leaf1, dst_addr1, 3);
      leaf2 = ip4_fib_mtrie_lookup_step (mtrie2, leaf2, dst_addr2, 3);
      leaf3 = ip4_fib_mtrie_lookup_step (mtrie3, leaf3, dst_addr3, 3);

	  /* 获取lb索引 */
      lb_index0 = ip4_fib_mtrie_leaf_get_adj_index (leaf0);
      lb_index1 = ip4_fib_mtrie_leaf_get_adj_index (leaf1);
      lb_index2 = ip4_fib_mtrie_leaf_get_adj_index (leaf2);
      lb_index3 = ip4_fib_mtrie_leaf_get_adj_index (leaf3);

	  /* 获取lb指针(由lb索引转换而来) */ 
      ASSERT (lb_index0 && lb_index1 && lb_index2 && lb_index3);
      lb0 = load_balance_get (lb_index0);
      lb1 = load_balance_get (lb_index1);
      lb2 = load_balance_get (lb_index2);
      lb3 = load_balance_get (lb_index3);

	  /* 检测lb中的桶数，桶数是2的指数 */
      ASSERT (lb0->lb_n_buckets > 0);
      ASSERT (is_pow2 (lb0->lb_n_buckets));
      ASSERT (lb1->lb_n_buckets > 0);
      ASSERT (is_pow2 (lb1->lb_n_buckets));
      ASSERT (lb2->lb_n_buckets > 0);
      ASSERT (is_pow2 (lb2->lb_n_buckets));
      ASSERT (lb3->lb_n_buckets > 0);
      ASSERT (is_pow2 (lb3->lb_n_buckets));

	  /* 初始化数据包流hash为0 */
      /* Use flow hash to compute multipath adjacency. */
      hash_c0 = vnet_buffer (b[0])->ip.flow_hash = 0;
      hash_c1 = vnet_buffer (b[1])->ip.flow_hash = 0;
      hash_c2 = vnet_buffer (b[2])->ip.flow_hash = 0;
      hash_c3 = vnet_buffer (b[3])->ip.flow_hash = 0;
	  /* lb中没有桶 */ 
      if (PREDICT_FALSE (lb0->lb_n_buckets > 1))
	{
	  /* 计算流hash */
	  flow_hash_config0 = lb0->lb_hash_config;
	  hash_c0 = vnet_buffer (b[0])->ip.flow_hash =
	    ip4_compute_flow_hash (ip0, flow_hash_config0);
	  /* 获取dpo结构 */
	  dpo0 =
	    load_balance_get_fwd_bucket (lb0,
					 (hash_c0 &
					  (lb0->lb_n_buckets_minus_1)));
	}
	  /* lb中有桶 */
      else
	{
	  /* 获取第一个桶的dpo结构 */
	  dpo0 = load_balance_get_bucket_i (lb0, 0);
	}
	  /* 跟第1个数据包一样 */
      if (PREDICT_FALSE (lb1->lb_n_buckets > 1))
	{
	  flow_hash_config1 = lb1->lb_hash_config;
	  hash_c1 = vnet_buffer (b[1])->ip.flow_hash =
	    ip4_compute_flow_hash (ip1, flow_hash_config1);
	  dpo1 =
	    load_balance_get_fwd_bucket (lb1,
					 (hash_c1 &
					  (lb1->lb_n_buckets_minus_1)));
	}
      else
	{
	  dpo1 = load_balance_get_bucket_i (lb1, 0);
	}
	  /* 跟第1个数据包一样 */
      if (PREDICT_FALSE (lb2->lb_n_buckets > 1))
	{
	  flow_hash_config2 = lb2->lb_hash_config;
	  hash_c2 = vnet_buffer (b[2])->ip.flow_hash =
	    ip4_compute_flow_hash (ip2, flow_hash_config2);
	  dpo2 =
	    load_balance_get_fwd_bucket (lb2,
					 (hash_c2 &
					  (lb2->lb_n_buckets_minus_1)));
	}
      else
	{
	  dpo2 = load_balance_get_bucket_i (lb2, 0);
	}
	  /* 跟第1个数据包一样 */
      if (PREDICT_FALSE (lb3->lb_n_buckets > 1))
	{
	  flow_hash_config3 = lb3->lb_hash_config;
	  hash_c3 = vnet_buffer (b[3])->ip.flow_hash =
	    ip4_compute_flow_hash (ip3, flow_hash_config3);
	  dpo3 =
	    load_balance_get_fwd_bucket (lb3,
					 (hash_c3 &
					  (lb3->lb_n_buckets_minus_1)));
	}
      else
	{
	  dpo3 = load_balance_get_bucket_i (lb3, 0);
	}

	  /* 获取处理第1个数据包的下个node */
      next[0] = dpo0->dpoi_next_node;
	  /* 第1个数据的发送临接索引 */
      vnet_buffer (b[0])->ip.adj_index[VLIB_TX] = dpo0->dpoi_index;
	  /* 获取处理第2个数据包的下个node */
      next[1] = dpo1->dpoi_next_node;
	  /* 第2个数据的发送临接索引 */
      vnet_buffer (b[1])->ip.adj_index[VLIB_TX] = dpo1->dpoi_index;
	  /* 获取处理第3个数据包的下个node */
      next[2] = dpo2->dpoi_next_node;
	  /* 第3个数据的发送临接索引 */
      vnet_buffer (b[2])->ip.adj_index[VLIB_TX] = dpo2->dpoi_index;
	  /* 获取处理第4个数据包的下个node */
      next[3] = dpo3->dpoi_next_node;
	  /* 第4个数据的发送临接索引 */
      vnet_buffer (b[3])->ip.adj_index[VLIB_TX] = dpo3->dpoi_index;

	  /* 更新lb相关联计数器值，包括cpu index, packets，bytes等 */
      vlib_increment_combined_counter
	(cm, thread_index, lb_index0, 1,
	 vlib_buffer_length_in_chain (vm, b[0]));
      vlib_increment_combined_counter
	(cm, thread_index, lb_index1, 1,
	 vlib_buffer_length_in_chain (vm, b[1]));
      vlib_increment_combined_counter
	(cm, thread_index, lb_index2, 1,
	 vlib_buffer_length_in_chain (vm, b[2]));
      vlib_increment_combined_counter
	(cm, thread_index, lb_index3, 1,
	 vlib_buffer_length_in_chain (vm, b[3]));

	  /* 数据包指针后移4 */
      b += 4;
	  /* nexts中数组索引加4 */
      next += 4;
	  /* frame中数据包个数减4 */
      n_left -= 4;
    }
/* 跟上述缓存行处理一样*/
#elif (CLIB_N_PREFETCHES >= 4)
  while (n_left >= 4)
    {
      ip4_header_t *ip0, *ip1;
      const load_balance_t *lb0, *lb1;
      ip4_fib_mtrie_t *mtrie0, *mtrie1;
      ip4_fib_mtrie_leaf_t leaf0, leaf1;
      ip4_address_t *dst_addr0, *dst_addr1;
      u32 lb_index0, lb_index1;
      flow_hash_config_t flow_hash_config0, flow_hash_config1;
      u32 hash_c0, hash_c1;
      const dpo_id_t *dpo0, *dpo1;

      /* Prefetch next iteration. */
      {
	vlib_prefetch_buffer_header (b[2], LOAD);
	vlib_prefetch_buffer_header (b[3], LOAD);

	CLIB_PREFETCH (b[2]->data, sizeof (ip0[0]), LOAD);
	CLIB_PREFETCH (b[3]->data, sizeof (ip0[0]), LOAD);
      }

      ip0 = vlib_buffer_get_current (b[0]);
      ip1 = vlib_buffer_get_current (b[1]);

      dst_addr0 = &ip0->dst_address;
      dst_addr1 = &ip1->dst_address;

      ip_lookup_set_buffer_fib_index (im->fib_index_by_sw_if_index, b[0]);
      ip_lookup_set_buffer_fib_index (im->fib_index_by_sw_if_index, b[1]);

      mtrie0 = &ip4_fib_get (vnet_buffer (b[0])->ip.fib_index)->mtrie;
      mtrie1 = &ip4_fib_get (vnet_buffer (b[1])->ip.fib_index)->mtrie;

      leaf0 = ip4_fib_mtrie_lookup_step_one (mtrie0, dst_addr0);
      leaf1 = ip4_fib_mtrie_lookup_step_one (mtrie1, dst_addr1);

      leaf0 = ip4_fib_mtrie_lookup_step (mtrie0, leaf0, dst_addr0, 2);
      leaf1 = ip4_fib_mtrie_lookup_step (mtrie1, leaf1, dst_addr1, 2);

      leaf0 = ip4_fib_mtrie_lookup_step (mtrie0, leaf0, dst_addr0, 3);
      leaf1 = ip4_fib_mtrie_lookup_step (mtrie1, leaf1, dst_addr1, 3);

      lb_index0 = ip4_fib_mtrie_leaf_get_adj_index (leaf0);
      lb_index1 = ip4_fib_mtrie_leaf_get_adj_index (leaf1);

      ASSERT (lb_index0 && lb_index1);
      lb0 = load_balance_get (lb_index0);
      lb1 = load_balance_get (lb_index1);

      ASSERT (lb0->lb_n_buckets > 0);
      ASSERT (is_pow2 (lb0->lb_n_buckets));
      ASSERT (lb1->lb_n_buckets > 0);
      ASSERT (is_pow2 (lb1->lb_n_buckets));

      /* Use flow hash to compute multipath adjacency. */
      hash_c0 = vnet_buffer (b[0])->ip.flow_hash = 0;
      hash_c1 = vnet_buffer (b[1])->ip.flow_hash = 0;
      if (PREDICT_FALSE (lb0->lb_n_buckets > 1))
	{
	  flow_hash_config0 = lb0->lb_hash_config;
	  hash_c0 = vnet_buffer (b[0])->ip.flow_hash =
	    ip4_compute_flow_hash (ip0, flow_hash_config0);
	  dpo0 =
	    load_balance_get_fwd_bucket (lb0,
					 (hash_c0 &
					  (lb0->lb_n_buckets_minus_1)));
	}
      else
	{
	  dpo0 = load_balance_get_bucket_i (lb0, 0);
	}
      if (PREDICT_FALSE (lb1->lb_n_buckets > 1))
	{
	  flow_hash_config1 = lb1->lb_hash_config;
	  hash_c1 = vnet_buffer (b[1])->ip.flow_hash =
	    ip4_compute_flow_hash (ip1, flow_hash_config1);
	  dpo1 =
	    load_balance_get_fwd_bucket (lb1,
					 (hash_c1 &
					  (lb1->lb_n_buckets_minus_1)));
	}
      else
	{
	  dpo1 = load_balance_get_bucket_i (lb1, 0);
	}

      next[0] = dpo0->dpoi_next_node;
      vnet_buffer (b[0])->ip.adj_index[VLIB_TX] = dpo0->dpoi_index;
      next[1] = dpo1->dpoi_next_node;
      vnet_buffer (b[1])->ip.adj_index[VLIB_TX] = dpo1->dpoi_index;

      vlib_increment_combined_counter
	(cm, thread_index, lb_index0, 1,
	 vlib_buffer_length_in_chain (vm, b[0]));
      vlib_increment_combined_counter
	(cm, thread_index, lb_index1, 1,
	 vlib_buffer_length_in_chain (vm, b[1]));

      b += 2;
      next += 2;
      n_left -= 2;
    }
#endif
/* 跟上述缓存行处理一样*/
  while (n_left > 0)
    {
      ip4_header_t *ip0;
      const load_balance_t *lb0;
      ip4_fib_mtrie_t *mtrie0;
      ip4_fib_mtrie_leaf_t leaf0;
      ip4_address_t *dst_addr0;
      u32 lbi0;
      flow_hash_config_t flow_hash_config0;
      const dpo_id_t *dpo0;
      u32 hash_c0;

      ip0 = vlib_buffer_get_current (b[0]);
      dst_addr0 = &ip0->dst_address;
      ip_lookup_set_buffer_fib_index (im->fib_index_by_sw_if_index, b[0]);

      mtrie0 = &ip4_fib_get (vnet_buffer (b[0])->ip.fib_index)->mtrie;
      leaf0 = ip4_fib_mtrie_lookup_step_one (mtrie0, dst_addr0);
      leaf0 = ip4_fib_mtrie_lookup_step (mtrie0, leaf0, dst_addr0, 2);
      leaf0 = ip4_fib_mtrie_lookup_step (mtrie0, leaf0, dst_addr0, 3);
      lbi0 = ip4_fib_mtrie_leaf_get_adj_index (leaf0);

      ASSERT (lbi0);
      lb0 = load_balance_get (lbi0);

      ASSERT (lb0->lb_n_buckets > 0);
      ASSERT (is_pow2 (lb0->lb_n_buckets));

      /* Use flow hash to compute multipath adjacency. */
      hash_c0 = vnet_buffer (b[0])->ip.flow_hash = 0;
      if (PREDICT_FALSE (lb0->lb_n_buckets > 1))
	{
	  flow_hash_config0 = lb0->lb_hash_config;

	  hash_c0 = vnet_buffer (b[0])->ip.flow_hash =
	    ip4_compute_flow_hash (ip0, flow_hash_config0);
	  dpo0 =
	    load_balance_get_fwd_bucket (lb0,
					 (hash_c0 &
					  (lb0->lb_n_buckets_minus_1)));
	}
      else
	{
	  dpo0 = load_balance_get_bucket_i (lb0, 0);
	}

      next[0] = dpo0->dpoi_next_node;
      vnet_buffer (b[0])->ip.adj_index[VLIB_TX] = dpo0->dpoi_index;

      vlib_increment_combined_counter (cm, thread_index, lbi0, 1,
				       vlib_buffer_length_in_chain (vm,
								    b[0]));

      b += 1;
      next += 1;
      n_left -= 1;
    }

  vlib_buffer_enqueue_to_next (vm, node, from, nexts, frame->n_vectors);

  if (node->flags & VLIB_NODE_FLAG_TRACE)
    ip4_forward_next_trace (vm, node, frame, VLIB_TX);

  return frame->n_vectors;
}
```

