## adj

adj模块用于管理二层next-hop信息、rewrite信息。数据平面。

### 查询

#### vnet/adj/adj.c

```
VLIB_CLI_COMMAND (adj_show_command, static) = {
    .path = "show adj",
    .short_help = "show adj [<adj_index>] [interface] [summary]",
    .function = adj_show,
};
```

```
static clib_error_t *
adj_show (vlib_main_t * vm,
	  unformat_input_t * input,
	  vlib_cli_command_t * cmd)
{
    /* adj索引 */
    adj_index_t ai = ADJ_INDEX_INVALID;
    u32 sw_if_index = ~0;
    int summary = 0;

    while (unformat_check_input (input) != UNFORMAT_END_OF_INPUT)
    {
	/* 解析adj索引 */
	if (unformat (input, "%d", &ai))
	    ;
	/* 解析是否要显示摘要 */
	else if (unformat (input, "sum"))
	    summary = 1;
	else if (unformat (input, "summary"))
	    summary = 1;
	/* 解析接口 */
	else if (unformat (input, "%U",
			   unformat_vnet_sw_interface, vnet_get_main(),
			   &sw_if_index))
	    ;
	/* 其他 */
	else
	    break;
    }

	/* 要显示摘要 */
    if (summary)
    {
	    /* 显示adj总个数 */
        vlib_cli_output (vm, "Number of adjacencies: %d", pool_elts(adj_pool));
		/* adj的计数器开关情况，enabled or disabled */
        vlib_cli_output (vm, "Per-adjacency counters: %s",
                         (adj_are_counters_enabled() ?
                          "enabled":
                          "disabled"));
    }
	/* 不显示摘要 */
    else
    {
	    /* 指定了adj索引 */
        if (ADJ_INDEX_INVALID != ai)
        {
			/* 判断指定的adj是否被释放（free） */
            if (pool_is_free_index(adj_pool, ai))
            {
			    /* adj已被释放，无效，返回错误 */
                vlib_cli_output (vm, "adjacency %d invalid", ai);
                return 0;
            }
			
			/* [@0] ipv4-glean: xge0/0/2: mtu:9000 ffffffffffff001b21bd061d0806
			显示adj，带详细，包括adj索引、下个动作索引、接口、mtu、mac字符串等 */
            vlib_cli_output (vm, "[@%d] %U",
                             ai,
                             format_ip_adjacency,  ai,
                             FORMAT_IP_ADJACENCY_DETAIL);
        }
		/* 未指定adj索引 */
        else
        {
            /* *INDENT-OFF* */
			/* 遍历，查找接口相等的adj */
            pool_foreach_index(ai, adj_pool,
            ({
                if (~0 != sw_if_index &&
                    sw_if_index != adj_get_sw_if_index(ai))
                {
                }
				/* 找到adj，接口相等 */
                else
                {
				    /* 显示adj，不带详细 */
                    vlib_cli_output (vm, "[@%d] %U",
                                     ai,
                                     format_ip_adjacency, ai,
                                     FORMAT_IP_ADJACENCY_NONE);
                }
            }));
            /* *INDENT-ON* */
        }
    }
    return 0;
}
```

