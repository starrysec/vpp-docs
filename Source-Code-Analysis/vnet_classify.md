## classify分类器

VPP分类器核心代码位于`vnet/classify`目录，用于实现ACL、Filter、策略路由、QoS、镜像等需要进行数据包分类操作的模块。

### 分类表
分类表(classify table)配置和查看代码位于`vnet/classify/vnet_classify.c`中。

#### 创建/删除分类表
```
VLIB_CLI_COMMAND (classify_table, static) =
{
  .path = "classify table",
  .short_help =
  "classify table [miss-next|l2-miss_next|acl-miss-next <next_index>]"
  "\n mask <mask-value> buckets <nn> [skip <n>] [match <n>]"
  "\n [current-data-flag <n>] [current-data-offset <n>] [table <n>]"
  "\n [memory-size <nn>[M][G]] [next-table <n>]"
  "\n [del] [del-chain]",
  .function = classify_table_command_fn,
};

static clib_error_t *
classify_table_command_fn (vlib_main_t * vm,
			   unformat_input_t * input, vlib_cli_command_t * cmd)
{
  u32 nbuckets = 2;
  u32 skip = ~0;
  u32 match = ~0;
  int is_add = 1;
  int del_chain = 0;
  u32 table_index = ~0;
  u32 next_table_index = ~0;
  u32 miss_next_index = ~0;
  u32 memory_size = 2 << 20;
  u32 tmp;
  u32 current_data_flag = 0;
  int current_data_offset = 0;

  u8 *mask = 0;
  vnet_classify_main_t *cm = &vnet_classify_main;
  int rv;

  while (unformat_check_input (input) != UNFORMAT_END_OF_INPUT)
  {
	  // 添加还是删除分类表、分类表链
      if (unformat (input, "del"))
		is_add = 0;
      else if (unformat (input, "del-chain"))
	  {
		is_add = 0;
		del_chain = 1;
	  }
	  // table中buckets（vnet_classify_bucket_t）个数个数
      else if (unformat (input, "buckets %d", &nbuckets))
	  ;
	  // 
      else if (unformat (input, "skip %d", &skip))
	  ;
	  // 
      else if (unformat (input, "match %d", &match))
	  ;
	  // 指定table index，未指定则为add，指定了则为update
      else if (unformat (input, "table %d", &table_index))
	  ;
	  // 
      else if (unformat (input, "mask %U", unformat_classify_mask,
			 &mask, &skip, &match))
	  ;
	  // table允许的最大内存，单位M/G
      else if (unformat (input, "memory-size %uM", &tmp))
	    memory_size = tmp << 20;
      else if (unformat (input, "memory-size %uG", &tmp))
	    memory_size = tmp << 30;
	  // 下个table的index
      else if (unformat (input, "next-table %d", &next_table_index))
	    ;
	  // 如果next_table_index为0，则走miss-next table
      else if (unformat (input, "miss-next %U", unformat_ip_next_index,
			 &miss_next_index))
	  ;
	  // 
      else if (unformat
	    (input, "l2-input-miss-next %U", unformat_l2_input_next_index,
	     &miss_next_index))
	  ;
	  // 
      else if (unformat
	      (input, "l2-output-miss-next %U", unformat_l2_output_next_index,
	     &miss_next_index))
	  ;
	  // 
      else if (unformat (input, "acl-miss-next %U", unformat_acl_next_index,
			 &miss_next_index))
      ;
	  // 
      else if (unformat (input, "current-data-flag %d", &current_data_flag))
	  ;
	  // 
      else if (unformat (input, "current-data-offset %d", &current_data_offset))
	  ;
      else
		break;
  }

  if (is_add && mask == 0 && table_index == ~0)
    return clib_error_return (0, "Mask required");

  if (is_add && skip == ~0 && table_index == ~0)
    return clib_error_return (0, "skip count required");

  if (is_add && match == ~0 && table_index == ~0)
    return clib_error_return (0, "match count required");

  if (!is_add && table_index == ~0)
    return clib_error_return (0, "table index required for delete");

  rv = vnet_classify_add_del_table (cm, mask, nbuckets, (u32) memory_size,
				    skip, match, next_table_index,
				    miss_next_index, &table_index,
				    current_data_flag, current_data_offset,
				    is_add, del_chain);
  switch (rv)
  {
    case 0:
      break;

    default:
      return clib_error_return (0, "vnet_classify_add_del_table returned %d",
				rv);
  }
  return 0;
}

int
vnet_classify_add_del_table (vnet_classify_main_t * cm,
			     u8 * mask,
			     u32 nbuckets,
			     u32 memory_size,
			     u32 skip,
			     u32 match,
			     u32 next_table_index,
			     u32 miss_next_index,
			     u32 * table_index,
			     u8 current_data_flag,
			     i16 current_data_offset,
			     int is_add, int del_chain)
{
  vnet_classify_table_t *t;

  if (is_add)
    {
      if (*table_index == ~0)	/* add */
	{
	  if (memory_size == 0)
	    return VNET_API_ERROR_INVALID_MEMORY_SIZE;

	  if (nbuckets == 0)
	    return VNET_API_ERROR_INVALID_VALUE;

	  if (match < 1 || match > 5)
	    return VNET_API_ERROR_INVALID_VALUE;

	  t = vnet_classify_new_table (cm, mask, nbuckets, memory_size,
				       skip, match);
	  t->next_table_index = next_table_index;
	  t->miss_next_index = miss_next_index;
	  t->current_data_flag = current_data_flag;
	  t->current_data_offset = current_data_offset;
	  *table_index = t - cm->tables;
	}
      else			/* update */
	{
	  vnet_classify_main_t *cm = &vnet_classify_main;
	  t = pool_elt_at_index (cm->tables, *table_index);

	  t->next_table_index = next_table_index;
	}
      return 0;
    }

  vnet_classify_delete_table_index (cm, *table_index, del_chain);
  return 0;
}
```

#### 查看分配表

### 分类会话
分类会话（classify session）即分类规则，规则被添加到指定的分类表中。

### 在接口使能
### 在flow使能