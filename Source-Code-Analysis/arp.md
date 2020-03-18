## arp

### arp弧和特性

#### vnet/arp/arp.c

```
VNET_FEATURE_ARC_INIT (arp_feat, static) =
{
  .arc_name = "arp",
  .start_nodes = VNET_FEATURES ("arp-input"),
  .last_in_arc = "error-drop",
  .arc_index_ptr = &ethernet_arp_main.feature_arc_index,
};
```

```
VNET_FEATURE_INIT (arp_reply_feat_node, static) =
{
  .arc_name = "arp",
  .node_name = "arp-reply",
  .runs_before = VNET_FEATURES ("arp-disabled"),
};

VNET_FEATURE_INIT (arp_proxy_feat_node, static) =
{
  .arc_name = "arp",
  .node_name = "arp-proxy",
  .runs_after = VNET_FEATURES ("arp-reply"),
  .runs_before = VNET_FEATURES ("arp-disabled"),
};

VNET_FEATURE_INIT (arp_disabled_feat_node, static) =
{
  .arc_name = "arp",
  .node_name = "arp-disabled",
  .runs_before = VNET_FEATURES ("error-drop"),
};

VNET_FEATURE_INIT (arp_drop_feat_node, static) =
{
  .arc_name = "arp",
  .node_name = "error-drop",
  .runs_before = 0,	/* last feature */
};
```

### arp图节点

#### vnet/arp/arp.c

```
VLIB_REGISTER_NODE (arp_input_node, static) =
{
  .function = arp_input,
  .name = "arp-input",
  .vector_size = sizeof (u32),
  .n_errors = ETHERNET_ARP_N_ERROR,
  .error_strings = ethernet_arp_error_strings,
  .n_next_nodes = ARP_INPUT_N_NEXT,
  .next_nodes = {
    [ARP_INPUT_NEXT_DROP] = "error-drop",
    [ARP_INPUT_NEXT_DISABLED] = "arp-disabled",
  },
  .format_buffer = format_ethernet_arp_header,
  .format_trace = format_ethernet_arp_input_trace,
};

VLIB_REGISTER_NODE (arp_disabled_node, static) =
{
  .function = arp_disabled,
  .name = "arp-disabled",
  .vector_size = sizeof (u32),
  .n_errors = ARP_DISABLED_N_ERROR,
  .error_strings = arp_disabled_error_strings,
  .n_next_nodes = ARP_DISABLED_N_NEXT,
  .next_nodes = {
    [ARP_INPUT_NEXT_DROP] = "error-drop",
  },
  .format_buffer = format_ethernet_arp_header,
  .format_trace = format_ethernet_arp_input_trace,
};

VLIB_REGISTER_NODE (arp_reply_node, static) =
{
  .function = arp_reply,
  .name = "arp-reply",
  .vector_size = sizeof (u32),
  .n_errors = ETHERNET_ARP_N_ERROR,
  .error_strings = ethernet_arp_error_strings,
  .n_next_nodes = ARP_REPLY_N_NEXT,
  .next_nodes = {
    [ARP_REPLY_NEXT_DROP] = "error-drop",
    [ARP_REPLY_NEXT_REPLY_TX] = "interface-output",
  },
  .format_buffer = format_ethernet_arp_header,
  .format_trace = format_ethernet_arp_input_trace,
};
```

#### vnet/arp/arp_proxy.c

```

VLIB_REGISTER_NODE (arp_proxy_node, static) =
{
  .function = arp_proxy,.name = "arp-proxy",.vector_size =
    sizeof (u32),.n_errors = ETHERNET_ARP_N_ERROR,.error_strings =
    ethernet_arp_error_strings,.n_next_nodes = ARP_REPLY_N_NEXT,.next_nodes =
  {
  [ARP_REPLY_NEXT_DROP] = "error-drop",
      [ARP_REPLY_NEXT_REPLY_TX] = "interface-output",}
,.format_buffer = format_ethernet_arp_header,.format_trace =
    format_ethernet_arp_input_trace,};

static clib_error_t *
show_ip4_arp (vlib_main_t * vm,
	      unformat_input_t * input, vlib_cli_command_t * cmd)
{
  arp_proxy_main_t *am = &arp_proxy_main;
  ethernet_proxy_arp_t *pa;

  if (vec_len (am->proxy_arps))
    {
      vlib_cli_output (vm, "Proxy arps enabled for:");
      vec_foreach (pa, am->proxy_arps)
      {
	vlib_cli_output (vm, "Fib_index %d   %U - %U ",
			 pa->fib_index,
			 format_ip4_address, &pa->lo_addr,
			 format_ip4_address, &pa->hi_addr);
      }
    }

  return (NULL);
}
```

