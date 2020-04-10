## Load Balancer

LB插件为VPP提供了负载均衡功能，它的灵感来自于Google MagLev：http://research.google.com/pubs/pub44824.html

负载均衡器配置了一系列虚拟IP（Virtual IPs, VIPs, 可以是带前缀形式），对于每个虚拟IP，对应一系列应用服务器（Application Server addresses, ASs）地址。

在给定的VIP（或者VIP前缀）上收到流量后，通过GRE隧道将流量发往不同的AS（同时会保证同一个会话发往同一个AS）。

VIP和AS都可以使用IPv4地址或者IPv6地址，但是对于给定的VIP，所有AS必须使用同一种封装类型（IPv4+GRE或者IPv6+GRE），即对于给定的VIP，所有AS地址必须是同一个地址簇（Address Family）。

**性能**

LB已经在1百万流的情况下进行了测试，每个核可以达到3Mpps。3Mpps已经非常不错了，在后续版本中还会持续优化。

### 配置

源文件位于`plugins/lb/cli.c`。

**设置全局LB参数**
```
VLIB_CLI_COMMAND (lb_conf_command, static) =
{
  .path = "lb conf",
  .short_help = "lb conf [ip4-src-address <addr>] [ip6-src-address <addr>] [buckets <n>] [timeout <s>]",
  .function = lb_conf_command_fn,
};

static clib_error_t *
lb_conf_command_fn (vlib_main_t * vm,
              unformat_input_t * input, vlib_cli_command_t * cmd)
{
  lb_main_t *lbm = &lb_main;
  unformat_input_t _line_input, *line_input = &_line_input;
  ip4_address_t ip4 = lbm->ip4_src_address;
  ip6_address_t ip6 = lbm->ip6_src_address;
  u32 per_cpu_sticky_buckets = lbm->per_cpu_sticky_buckets;
  u32 per_cpu_sticky_buckets_log2 = 0;
  u32 flow_timeout = lbm->flow_timeout;
  int ret;
  clib_error_t *error = 0;

  if (!unformat_user (input, unformat_line_input, line_input))
    return 0;

  while (unformat_check_input (line_input) != UNFORMAT_END_OF_INPUT)
  {
    // 解析用于封装GRE的源IPv4地址
    if (unformat(line_input, "ip4-src-address %U", unformat_ip4_address, &ip4))
      ;
    // 解析用于封装GRE的源IPv6地址
    else if (unformat(line_input, "ip6-src-address %U", unformat_ip6_address, &ip6))
      ;
    // 每个线程建立连接表的存储桶数
    else if (unformat(line_input, "buckets %d", &per_cpu_sticky_buckets))
      ;
    // 根据buckets计算buckets-log2的值，用以修正buckets为2的指数
    else if (unformat(line_input, "buckets-log2 %d", &per_cpu_sticky_buckets_log2)) {
      if (per_cpu_sticky_buckets_log2 >= 32)
        return clib_error_return (0, "buckets-log2 value is too high");
      per_cpu_sticky_buckets = 1 << per_cpu_sticky_buckets_log2;
    } 
    // 一个已建立的连接，在流上没有接收到数据包的情况下，在连接表中保留的秒数
    else if (unformat(line_input, "timeout %d", &flow_timeout))
      ;
    else {
      error = clib_error_return (0, "parse error: '%U'",
                                 format_unformat_error, line_input);
      goto done;
    }
  }

  // 垃圾回收
  lb_garbage_collection();

  // 设置lb全局参数
  if ((ret = lb_conf(&ip4, &ip6, per_cpu_sticky_buckets, flow_timeout))) {
    error = clib_error_return (0, "lb_conf error %d", ret);
    goto done;
  }

done:
  unformat_free (line_input);

  return error;
}
```

**设置VIP**

```
lb vip 2002::/16 encap gre6 new_len 1024
lb vip 2003::/16 encap gre4 new_len 2048
lb vip 80.0.0.0/8 encap gre6 new_len 16
lb vip 90.0.0.0/8 encap gre4 new_len 1024
```

```
VLIB_CLI_COMMAND (lb_vip_command, static) =
{
  .path = "lb vip",
  .short_help = "lb vip <prefix> "
      "[protocol (tcp|udp) port <n>] "
      "[encap (gre6|gre4|l3dsr|nat4|nat6)] "
      "[dscp <n>] "
      "[type (nodeport|clusterip) target_port <n>] "
      "[new_len <n>] [del]",
  .function = lb_vip_command_fn,
};

static clib_error_t *
lb_vip_command_fn (vlib_main_t * vm,
              unformat_input_t * input, vlib_cli_command_t * cmd)
{
  unformat_input_t _line_input, *line_input = &_line_input;
  lb_vip_add_args_t args;
  u8 del = 0;
  int ret;
  u32 port = 0;
  u32 encap = 0;
  u32 dscp = ~0;
  u32 srv_type = LB_SRV_TYPE_CLUSTERIP;
  u32 target_port = 0;
  clib_error_t *error = 0;

  args.new_length = 1024;

  if (!unformat_user (input, unformat_line_input, line_input))
    return 0;

  // 解析vip或者vip前缀
  if (!unformat(line_input, "%U", unformat_ip46_prefix, &(args.prefix),
                &(args.plen), IP46_TYPE_ANY, &(args.plen))) {
    error = clib_error_return (0, "invalid vip prefix: '%U'",
                               format_unformat_error, line_input);
    goto done;
  }

  while (unformat_check_input (line_input) != UNFORMAT_END_OF_INPUT)
  {
    // new_len是新连接表的大小。它应比VIP的AS数量大1或2个数量级，以确保良好的负载平衡。
    if (unformat(line_input, "new_len %d", &(args.new_length)))
      ;
    // 新建or删除
    else if (unformat(line_input, "del"))
      del = 1;
    // 采用的传输协议是tcp or udp
    else if (unformat(line_input, "protocol tcp"))
      {
        args.protocol = (u8)IP_PROTOCOL_TCP;
      }
    else if (unformat(line_input, "protocol udp"))
      {
        args.protocol = (u8)IP_PROTOCOL_UDP;
      }
    // 监听的vip的port，0为所有ports
    else if (unformat(line_input, "port %d", &port))
      ;
    // 封装形式：gre4, gre6, l3dsr, nat4, nat6
    else if (unformat(line_input, "encap gre4"))
      encap = LB_ENCAP_TYPE_GRE4;
    else if (unformat(line_input, "encap gre6"))
      encap = LB_ENCAP_TYPE_GRE6;
    else if (unformat(line_input, "encap l3dsr"))
      encap = LB_ENCAP_TYPE_L3DSR;
    else if (unformat(line_input, "encap nat4"))
      encap = LB_ENCAP_TYPE_NAT4;
    else if (unformat(line_input, "encap nat6"))
      encap = LB_ENCAP_TYPE_NAT6;
    // 设置差分服务代码点（dscp）
    else if (unformat(line_input, "dscp %d", &dscp))
      ;
    // lb支持负载后端是k8s时的两种形式：ClusterIP和NodePort，参考lv_vip_encap_args_t
    else if (unformat(line_input, "type clusterip"))
      srv_type = LB_SRV_TYPE_CLUSTERIP;
    else if (unformat(line_input, "type nodeport"))
      srv_type = LB_SRV_TYPE_NODEPORT;
    // k8s中与特定service通信的Pod的port，参考lv_vip_encap_args_t
    else if (unformat(line_input, "target_port %d", &target_port))
      ;
    else {
      error = clib_error_return (0, "parse error: '%U'",
                                format_unformat_error, line_input);
      goto done;
    }
  }

  /* if port == 0, it means all-port VIP */
  if (port == 0)
    {
      args.protocol = ~0;
      args.port = 0;
    }
  else
    {
      args.port = (u16)port;
    }

  if ((encap != LB_ENCAP_TYPE_L3DSR) && (dscp != ~0))
    {
      error = clib_error_return(0, "lb_vip_add error: "
                                "should not configure dscp for none L3DSR.");
      goto done;
    }

  if ((encap == LB_ENCAP_TYPE_L3DSR) && (dscp >= 64))
    {
      error = clib_error_return(0, "lb_vip_add error: "
                                "dscp for L3DSR should be less than 64.");
      goto done;
    }

  // vip为ipv4
  if (ip46_prefix_is_ip4(&(args.prefix), (args.plen)))
    {
      // 封装类型
      if (encap == LB_ENCAP_TYPE_GRE4)
        args.type = LB_VIP_TYPE_IP4_GRE4;
      else if (encap == LB_ENCAP_TYPE_GRE6)
        args.type = LB_VIP_TYPE_IP4_GRE6;
      else if (encap == LB_ENCAP_TYPE_L3DSR)
        args.type = LB_VIP_TYPE_IP4_L3DSR;
      else if (encap == LB_ENCAP_TYPE_NAT4)
        args.type = LB_VIP_TYPE_IP4_NAT4;
      else if (encap == LB_ENCAP_TYPE_NAT6)
        {
          error = clib_error_return(0, "currently does not support NAT46");
          goto done;
        }
    }
  // vip为ipv6
  else
    {
       // 封装类型
      if (encap == LB_ENCAP_TYPE_GRE4)
        args.type = LB_VIP_TYPE_IP6_GRE4;
      else if (encap == LB_ENCAP_TYPE_GRE6)
        args.type = LB_VIP_TYPE_IP6_GRE6;
      else if (encap == LB_ENCAP_TYPE_NAT6)
        args.type = LB_VIP_TYPE_IP6_NAT6;
      else if (encap == LB_ENCAP_TYPE_NAT4)
        {
          error = clib_error_return(0, "currently does not support NAT64");
          goto done;
        }
    }

  // 垃圾回收
  lb_garbage_collection();

  u32 index;
  // 新建
  if (!del) {
    // DSR
    if (encap == LB_ENCAP_TYPE_L3DSR) {
      args.encap_args.dscp = (u8)(dscp & 0x3F);
    }
    // NAT
    else if ((encap == LB_ENCAP_TYPE_NAT4)
               || (encap == LB_ENCAP_TYPE_NAT6))
    {
      args.encap_args.srv_type = (u8) srv_type;
      args.encap_args.target_port = (u16) target_port;
    }

    // 添加vip
    if ((ret = lb_vip_add(args, &index))) {
      error = clib_error_return (0, "lb_vip_add error %d", ret);
      goto done;
    } else {
      vlib_cli_output(vm, "lb_vip_add ok %d", index);
    }
  // 删除
  } else {
    // 按index删除
    if ((ret = lb_vip_find_index(&(args.prefix), args.plen,
                                 args.protocol, args.port, &index))) {
      error = clib_error_return (0, "lb_vip_find_index error %d", ret);
      goto done;
    } else if ((ret = lb_vip_del(index))) {
      error = clib_error_return (0, "lb_vip_del error %d", ret);
      goto done;
    }
  }

done:
  unformat_free (line_input);

  return error;
}
```

**设置AS(对每个VIP)**

您可以一次添加（或删除）多个AS（对于单个VIP）。请注意，AS地址系列必须与VIP封装IP地址簇相对应。

```
lb as 2002::/16 2001::2 2001::3 2001::4
lb as 2003::/16 10.0.0.1 10.0.0.2
lb as 80.0.0.0/8 2001::2
lb as 90.0.0.0/8 10.0.0.1
```

```
VLIB_CLI_COMMAND (lb_as_command, static) =
{
  .path = "lb as",
  .short_help = "lb as <vip-prefix> [protocol (tcp|udp) port <n>]"
      " [<address> [<address> [...]]] [del] [flush]",
  .function = lb_as_command_fn,
};

static clib_error_t *
lb_as_command_fn (vlib_main_t * vm,
              unformat_input_t * input, vlib_cli_command_t * cmd)
{
  unformat_input_t _line_input, *line_input = &_line_input;
  ip46_address_t vip_prefix, as_addr;
  u8 vip_plen;
  ip46_address_t *as_array = 0;
  u32 vip_index;
  u32 port = 0;
  u8 protocol = 0;
  u8 del = 0;
  u8 flush = 0;
  int ret;
  clib_error_t *error = 0;

  if (!unformat_user (input, unformat_line_input, line_input))
    return 0;

  // 解析vip或vip前缀
  if (!unformat(line_input, "%U", unformat_ip46_prefix,
                &vip_prefix, &vip_plen, IP46_TYPE_ANY))
  {
    error = clib_error_return (0, "invalid as address: '%U'",
                               format_unformat_error, line_input);
    goto done;
  }

  while (unformat_check_input (line_input) != UNFORMAT_END_OF_INPUT)
  {
    // 解析AS地址
    if (unformat(line_input, "%U", unformat_ip46_address,
                 &as_addr, IP46_TYPE_ANY))
      {
        vec_add1(as_array, as_addr);
      }
    // add or delete
    else if (unformat(line_input, "del"))
      {
        del = 1;
      }
    // flush
    else if (unformat(line_input, "flush"))
      {
        flush = 1;
      }
    // 传输层协议，tcp or udp
    else if (unformat(line_input, "protocol tcp"))
      {
          protocol = (u8)IP_PROTOCOL_TCP;
      }
    else if (unformat(line_input, "protocol udp"))
      {
          protocol = (u8)IP_PROTOCOL_UDP;
      }
    // AS端口
    else if (unformat(line_input, "port %d", &port))
      ;
    else {
      error = clib_error_return (0, "parse error: '%U'",
                                 format_unformat_error, line_input);
      goto done;
    }
  }

  /* If port == 0, it means all-port VIP */
  if (port == 0)
    {
      protocol = ~0;
    }

  // find vip index
  if ((ret = lb_vip_find_index(&vip_prefix, vip_plen, protocol,
                               (u16)port, &vip_index))){
    error = clib_error_return (0, "lb_vip_find_index error %d", ret);
    goto done;
  }

  // 配置了至少一个as地址
  if (!vec_len(as_array)) {
    error = clib_error_return (0, "No AS address provided");
    goto done;
  }

  // 垃圾回收
  lb_garbage_collection();
  clib_warning("vip index is %d", vip_index);

  // 删除
  if (del) {
    // 删除指定vip的as地址
    if ((ret = lb_vip_del_ass(vip_index, as_array, vec_len(as_array), flush)))
    {
      error = clib_error_return (0, "lb_vip_del_ass error %d", ret);
      goto done;
    }
  // 新建
  } else {
    // 添加指定vip的as地址
    if ((ret = lb_vip_add_ass(vip_index, as_array, vec_len(as_array))))
    {
      error = clib_error_return (0, "lb_vip_add_ass error %d", ret);
      goto done;
    }
  }

done:
  unformat_free (line_input);
  vec_free(as_array);

  return error;
}
```

### 查询

```
VLIB_CLI_COMMAND (lb_show_command, static) =
{
  .path = "show lb",
  .short_help = "show lb",
  .function = lb_show_command_fn,
};

static clib_error_t *
lb_show_command_fn (vlib_main_t * vm,
              unformat_input_t * input, vlib_cli_command_t * cmd)
{
  vlib_cli_output(vm, "%U", format_lb_main);
  return NULL;
}

VLIB_CLI_COMMAND (lb_show_vips_command, static) =
{
  .path = "show lb vips",
  .short_help = "show lb vips [verbose]",
  .function = lb_show_vips_command_fn,
};

static clib_error_t *
lb_show_vips_command_fn (vlib_main_t * vm,
              unformat_input_t * input, vlib_cli_command_t * cmd)
{
  unformat_input_t line_input;
  lb_main_t *lbm = &lb_main;
  lb_vip_t *vip;
  u8 verbose = 0;

  if (!unformat_user (input, unformat_line_input, &line_input))
      return 0;

  if (unformat(&line_input, "verbose"))
    verbose = 1;

  /* Hide dummy VIP */
  pool_foreach(vip, lbm->vips, {
    if (vip != lbm->vips) {
      vlib_cli_output(vm, "%U\n", verbose?format_lb_vip_detailed:format_lb_vip, vip);
    }
  });

  unformat_free (&line_input);
  return NULL;
}
```

### 处理

源文件位于`plugins/lb/node.c`。

```
VLIB_REGISTER_NODE (lb4_gre4_node) =
  {
    .function = lb4_gre4_node_fn,
    .name = "lb4-gre4",
    .vector_size = sizeof(u32),
    .format_trace = format_lb_trace,
    .n_errors = LB_N_ERROR,
    .error_strings = lb_error_strings,
    .n_next_nodes = LB_N_NEXT,
    .next_nodes =
        { [LB_NEXT_DROP] = "error-drop" },
  };
  
static uword
lb4_gre4_node_fn (vlib_main_t * vm, vlib_node_runtime_t * node,
                  vlib_frame_t * frame)
{
  return lb_node_fn (vm, node, frame, 1, LB_ENCAP_TYPE_GRE4, 0);
}  
  
static_always_inline uword
lb_node_fn (vlib_main_t * vm,
            vlib_node_runtime_t * node,
            vlib_frame_t * frame,
            u8 is_input_v4, //Compile-time parameter stating that is input is v4 (or v6)
            lb_encap_type_t encap_type, //Compile-time parameter is GRE4/GRE6/L3DSR/NAT4/NAT6
            u8 per_port_vip) //Compile-time parameter stating that is per_port_vip or not
{
  lb_main_t *lbm = &lb_main;
  u32 n_left_from, *from, next_index, *to_next, n_left_to_next;
  u32 thread_index = vm->thread_index;
  u32 lb_time = lb_hash_time_now (vm);

  lb_hash_t *sticky_ht = lb_get_sticky_table (thread_index);
  from = vlib_frame_vector_args (frame);
  n_left_from = frame->n_vectors;
  next_index = node->cached_next_index;

  u32 nexthash0 = 0;
  u32 next_vip_idx0 = ~0;
  if (PREDICT_TRUE(n_left_from > 0))
    {
      vlib_buffer_t *p0 = vlib_get_buffer (vm, from[0]);
      lb_node_get_hash (lbm, p0, is_input_v4, &nexthash0,
                        &next_vip_idx0, per_port_vip);
    }

  while (n_left_from > 0)
    {
      vlib_get_next_frame(vm, node, next_index, to_next, n_left_to_next);
      while (n_left_from > 0 && n_left_to_next > 0)
        {
          u32 pi0;
          vlib_buffer_t *p0;
          lb_vip_t *vip0;
          u32 asindex0 = 0;
          u16 len0;
          u32 available_index0;
          u8 counter = 0;
          u32 hash0 = nexthash0;
          u32 vip_index0 = next_vip_idx0;
          u32 next0;

          if (PREDICT_TRUE(n_left_from > 1))
            {
              vlib_buffer_t *p1 = vlib_get_buffer (vm, from[1]);
              //Compute next hash and prefetch bucket
              lb_node_get_hash (lbm, p1, is_input_v4,
                                &nexthash0, &next_vip_idx0,
                                per_port_vip);
              lb_hash_prefetch_bucket (sticky_ht, nexthash0);
              //Prefetch for encap, next
              CLIB_PREFETCH(vlib_buffer_get_current (p1) - 64, 64, STORE);
            }

          if (PREDICT_TRUE(n_left_from > 2))
            {
              vlib_buffer_t *p2;
              p2 = vlib_get_buffer (vm, from[2]);
              /* prefetch packet header and data */
              vlib_prefetch_buffer_header(p2, STORE);
              CLIB_PREFETCH(vlib_buffer_get_current (p2), 64, STORE);
            }

          pi0 = to_next[0] = from[0];
          from += 1;
          n_left_from -= 1;
          to_next += 1;
          n_left_to_next -= 1;

          p0 = vlib_get_buffer (vm, pi0);

          vip0 = pool_elt_at_index(lbm->vips, vip_index0);

          if (is_input_v4)
            {
              ip4_header_t *ip40;
              ip40 = vlib_buffer_get_current (p0);
              len0 = clib_net_to_host_u16 (ip40->length);
            }
          else
            {
              ip6_header_t *ip60;
              ip60 = vlib_buffer_get_current (p0);
              len0 = clib_net_to_host_u16 (ip60->payload_length)
                  + sizeof(ip6_header_t);
            }

          lb_hash_get (sticky_ht, hash0,
                       vip_index0, lb_time,
                       &available_index0, &asindex0);

          if (PREDICT_TRUE(asindex0 != 0))
            {
              //Found an existing entry
              counter = LB_VIP_COUNTER_NEXT_PACKET;
            }
          else if (PREDICT_TRUE(available_index0 != ~0))
            {
              //There is an available slot for a new flow
              asindex0 =
                  vip0->new_flow_table[hash0 & vip0->new_flow_table_mask].as_index;
              counter = LB_VIP_COUNTER_FIRST_PACKET;
              counter = (asindex0 == 0) ? LB_VIP_COUNTER_NO_SERVER : counter;

              //TODO: There are race conditions with as0 and vip0 manipulation.
              //Configuration may be changed, vectors resized, etc...

              //Dereference previously used
              vlib_refcount_add (
                  &lbm->as_refcount, thread_index,
                  lb_hash_available_value (sticky_ht, hash0, available_index0),
                  -1);
              vlib_refcount_add (&lbm->as_refcount, thread_index, asindex0, 1);

              //Add sticky entry
              //Note that when there is no AS configured, an entry is configured anyway.
              //But no configured AS is not something that should happen
              lb_hash_put (sticky_ht, hash0, asindex0,
                           vip_index0,
                           available_index0, lb_time);
            }
          else
            {
              //Could not store new entry in the table
              asindex0 =
                  vip0->new_flow_table[hash0 & vip0->new_flow_table_mask].as_index;
              counter = LB_VIP_COUNTER_UNTRACKED_PACKET;
            }

          vlib_increment_simple_counter (
              &lbm->vip_counters[counter], thread_index,
              vip_index0,
              1);

          //Now let's encap
          if ((encap_type == LB_ENCAP_TYPE_GRE4)
              || (encap_type == LB_ENCAP_TYPE_GRE6))
            {
              gre_header_t *gre0;
              if (encap_type == LB_ENCAP_TYPE_GRE4) /* encap GRE4*/
                {
                  ip4_header_t *ip40;
                  vlib_buffer_advance (
                      p0, -sizeof(ip4_header_t) - sizeof(gre_header_t));
                  ip40 = vlib_buffer_get_current (p0);
                  gre0 = (gre_header_t *) (ip40 + 1);
                  ip40->src_address = lbm->ip4_src_address;
                  ip40->dst_address = lbm->ass[asindex0].address.ip4;
                  ip40->ip_version_and_header_length = 0x45;
                  ip40->ttl = 128;
                  ip40->fragment_id = 0;
                  ip40->flags_and_fragment_offset = 0;
                  ip40->length = clib_host_to_net_u16 (
                      len0 + sizeof(gre_header_t) + sizeof(ip4_header_t));
                  ip40->protocol = IP_PROTOCOL_GRE;
                  ip40->checksum = ip4_header_checksum (ip40);
                }
              else /* encap GRE6*/
                {
                  ip6_header_t *ip60;
                  vlib_buffer_advance (
                      p0, -sizeof(ip6_header_t) - sizeof(gre_header_t));
                  ip60 = vlib_buffer_get_current (p0);
                  gre0 = (gre_header_t *) (ip60 + 1);
                  ip60->dst_address = lbm->ass[asindex0].address.ip6;
                  ip60->src_address = lbm->ip6_src_address;
                  ip60->hop_limit = 128;
                  ip60->ip_version_traffic_class_and_flow_label =
                      clib_host_to_net_u32 (0x6 << 28);
                  ip60->payload_length = clib_host_to_net_u16 (
                      len0 + sizeof(gre_header_t));
                  ip60->protocol = IP_PROTOCOL_GRE;
                }

              gre0->flags_and_version = 0;
              gre0->protocol =
                  (is_input_v4) ?
                      clib_host_to_net_u16 (0x0800) :
                      clib_host_to_net_u16 (0x86DD);
            }
          else if (encap_type == LB_ENCAP_TYPE_L3DSR) /* encap L3DSR*/
            {
              ip4_header_t *ip40;
              tcp_header_t *th0;
              ip_csum_t csum;
              u32 old_dst, new_dst;
              u8 old_tos, new_tos;

              ip40 = vlib_buffer_get_current (p0);
              old_dst = ip40->dst_address.as_u32;
              new_dst = lbm->ass[asindex0].address.ip4.as_u32;
              ip40->dst_address.as_u32 = lbm->ass[asindex0].address.ip4.as_u32;
              /* Get and rewrite DSCP bit */
              old_tos = ip40->tos;
              new_tos = (u8) ((vip0->encap_args.dscp & 0x3F) << 2);
              ip40->tos = (u8) ((vip0->encap_args.dscp & 0x3F) << 2);

              csum = ip40->checksum;
              csum = ip_csum_update (csum, old_tos, new_tos,
                                     ip4_header_t,
                                     tos /* changed member */);
              csum = ip_csum_update (csum, old_dst, new_dst,
                                     ip4_header_t,
                                     dst_address /* changed member */);
              ip40->checksum = ip_csum_fold (csum);

              /* Recomputing L4 checksum after dst-IP modifying */
              th0 = ip4_next_header (ip40);
              th0->checksum = 0;
              th0->checksum = ip4_tcp_udp_compute_checksum (vm, p0, ip40);
            }
          else if ((encap_type == LB_ENCAP_TYPE_NAT4)
              || (encap_type == LB_ENCAP_TYPE_NAT6))
            {
              ip_csum_t csum;
              udp_header_t *uh;

              /* do NAT */
              if ((is_input_v4 == 1) && (encap_type == LB_ENCAP_TYPE_NAT4))
                {
                  /* NAT44 */
                  ip4_header_t *ip40;
                  u32 old_dst;
                  ip40 = vlib_buffer_get_current (p0);
                  uh = (udp_header_t *) (ip40 + 1);
                  old_dst = ip40->dst_address.as_u32;
                  ip40->dst_address = lbm->ass[asindex0].address.ip4;

                  csum = ip40->checksum;
                  csum = ip_csum_sub_even (csum, old_dst);
                  csum = ip_csum_add_even (
                      csum, lbm->ass[asindex0].address.ip4.as_u32);
                  ip40->checksum = ip_csum_fold (csum);

                  if (ip40->protocol == IP_PROTOCOL_UDP)
                    {
                      uh->dst_port = vip0->encap_args.target_port;
                      csum = uh->checksum;
                      csum = ip_csum_sub_even (csum, old_dst);
                      csum = ip_csum_add_even (
                          csum, lbm->ass[asindex0].address.ip4.as_u32);
                      uh->checksum = ip_csum_fold (csum);
                    }
                  else
                    {
                      asindex0 = 0;
                    }
                }
              else if ((is_input_v4 == 0) && (encap_type == LB_ENCAP_TYPE_NAT6))
                {
                  /* NAT66 */
                  ip6_header_t *ip60;
                  ip6_address_t old_dst;

                  ip60 = vlib_buffer_get_current (p0);
                  uh = (udp_header_t *) (ip60 + 1);

                  old_dst.as_u64[0] = ip60->dst_address.as_u64[0];
                  old_dst.as_u64[1] = ip60->dst_address.as_u64[1];
                  ip60->dst_address.as_u64[0] =
                      lbm->ass[asindex0].address.ip6.as_u64[0];
                  ip60->dst_address.as_u64[1] =
                      lbm->ass[asindex0].address.ip6.as_u64[1];

                  if (PREDICT_TRUE(ip60->protocol == IP_PROTOCOL_UDP))
                    {
                      uh->dst_port = vip0->encap_args.target_port;
                      csum = uh->checksum;
                      csum = ip_csum_sub_even (csum, old_dst.as_u64[0]);
                      csum = ip_csum_sub_even (csum, old_dst.as_u64[1]);
                      csum = ip_csum_add_even (
                          csum, lbm->ass[asindex0].address.ip6.as_u64[0]);
                      csum = ip_csum_add_even (
                          csum, lbm->ass[asindex0].address.ip6.as_u64[1]);
                      uh->checksum = ip_csum_fold (csum);
                    }
                  else
                    {
                      asindex0 = 0;
                    }
                }
            }
          next0 = lbm->ass[asindex0].dpo.dpoi_next_node;
          //Note that this is going to error if asindex0 == 0
          vnet_buffer (p0)->ip.adj_index[VLIB_TX] =
              lbm->ass[asindex0].dpo.dpoi_index;

          if (PREDICT_FALSE(p0->flags & VLIB_BUFFER_IS_TRACED))
            {
              lb_trace_t *tr = vlib_add_trace (vm, node, p0, sizeof(*tr));
              tr->as_index = asindex0;
              tr->vip_index = vip_index0;
            }

          //Enqueue to next
          vlib_validate_buffer_enqueue_x1(
              vm, node, next_index, to_next, n_left_to_next, pi0, next0);
        }
      vlib_put_next_frame (vm, node, next_index, n_left_to_next);
    }

  return frame->n_vectors;
}
```