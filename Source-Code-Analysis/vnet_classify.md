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
	  // skip match vector的前几个
      else if (unformat (input, "skip %d", &skip))
	  ;
	  // match vector数量
      else if (unformat (input, "match %d", &match))
	  ;
	  // 指定table index，未指定则为add，指定了则为update
      else if (unformat (input, "table %d", &table_index))
	  ;
	  // 指定匹配用的mask
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
	  // 以下miss-next都同上
      else if (unformat
	    (input, "l2-input-miss-next %U", unformat_l2_input_next_index,
	     &miss_next_index))
	  ;
      else if (unformat
	      (input, "l2-output-miss-next %U", unformat_l2_output_next_index,
	     &miss_next_index))
	  ;
      else if (unformat (input, "acl-miss-next %U", unformat_acl_next_index,
			 &miss_next_index))
      ;
	  /* option to use current node's packet payload
         as the starting point from where packets are classified,
         This option is only valid for L2/L3 input ACL for now.
         0: by default, classify data from the buffer's start location
         1: classify packets from VPP node’s current data pointer
	  */
      else if (unformat (input, "current-data-flag %d", &current_data_flag))
	  ;
	  /* a signed value to shift the start location of the packet to be classified
         For example, if input IP ACL node is used, L2 header’s first byte
         can be accessible by configuring current_data_offset to -14
         if there is no vlan tag.
         This is valid only if current_data_flag is set to 1.
	  */
      else if (unformat (input, "current-data-offset %d", &current_data_offset))
	  ;
      else
		break;
  }

  // 添加，必须指定mask
  if (is_add && mask == 0 && table_index == ~0)
    return clib_error_return (0, "Mask required");

  // 添加，必须指定skip count
  if (is_add && skip == ~0 && table_index == ~0)
    return clib_error_return (0, "skip count required");

  // 添加，必须指定match count
  if (is_add && match == ~0 && table_index == ~0)
    return clib_error_return (0, "match count required");

  // 删除，必须指定table index
  if (!is_add && table_index == ~0)
    return clib_error_return (0, "table index required for delete");

  // 创建/删除分类表核心函数
  rv = vnet_classify_add_del_table (cm, mask, nbuckets, (u32) memory_size,
				    skip, match, next_table_index,
				    miss_next_index, &table_index,
				    current_data_flag, current_data_offset,
				    is_add, del_chain);
  // 检查返回值，0-正确，其他-错误
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

  // 添加
  if (is_add)
  {
    // 添加
    if (*table_index == ~0)	/* add */
	{
	  if (memory_size == 0)
	    return VNET_API_ERROR_INVALID_MEMORY_SIZE;

	  if (nbuckets == 0)
	    return VNET_API_ERROR_INVALID_VALUE;

	  if (match < 1 || match > 5)
	    return VNET_API_ERROR_INVALID_VALUE;

      // 创建分类表
	  t = vnet_classify_new_table (cm, mask, nbuckets, memory_size,
				       skip, match);
	  // 赋值
	  t->next_table_index = next_table_index;
	  t->miss_next_index = miss_next_index;
	  t->current_data_flag = current_data_flag;
	  t->current_data_offset = current_data_offset;
	  // 获取分类表索引
	  *table_index = t - cm->tables;
	}
	// 更新
    else			/* update */
	{
	  vnet_classify_main_t *cm = &vnet_classify_main;
	  // 根据table index获取table
	  t = pool_elt_at_index (cm->tables, *table_index);
	  // 赋值
	  t->next_table_index = next_table_index;
	}
    return 0;
  }

  // 删除
  vnet_classify_delete_table_index (cm, *table_index, del_chain);
  return 0;
}

vnet_classify_table_t *
vnet_classify_new_table (vnet_classify_main_t * cm,
			 u8 * mask, u32 nbuckets, u32 memory_size,
			 u32 skip_n_vectors, u32 match_n_vectors)
{
  vnet_classify_table_t *t;
  void *oldheap;

  // 规格化nbuckets为2的整数倍
  nbuckets = 1 << (max_log2 (nbuckets));

  // 从内存池申请分类表
  pool_get_aligned (cm->tables, t, CLIB_CACHE_LINE_BYTES);
  // 初始化table内存
  clib_memset (t, 0, sizeof (*t));

  // Make sure vector is long enough for given index(no header, specified alignment)
  vec_validate_aligned (t->mask, match_n_vectors - 1, sizeof (u32x4));
  // Copy Mask, Mask to apply after skipping N vectors
  clib_memcpy_fast (t->mask, mask, match_n_vectors * sizeof (u32x4));

  // 初始化table各字段
  t->next_table_index = ~0;
  t->nbuckets = nbuckets;
  t->log2_nbuckets = max_log2 (nbuckets);
  t->match_n_vectors = match_n_vectors;
  t->skip_n_vectors = skip_n_vectors;
  t->entries_per_page = 2;

  // 申请堆内存
#if USE_DLMALLOC == 0
  t->mheap = mheap_alloc (0 /* use VM */ , memory_size);
#else
  t->mheap = create_mspace (memory_size, 1 /* locked */ );
  /* classifier requires the memory to be contiguous, so can not expand. */
  mspace_disable_expand (t->mheap);
#endif

  // Make sure vector is long enough for given index(no header, specified alignment)
  vec_validate_aligned (t->buckets, nbuckets - 1, CLIB_CACHE_LINE_BYTES);
  // set per cpu new heap, return old heap.
  oldheap = clib_mem_set_heap (t->mheap);
  // 给spinlock申请内存
  clib_spinlock_init (&t->writer_lock);
  // reset old heap.
  clib_mem_set_heap (oldheap);
  return (t);
}
```

#### 查看分配表

### 分类会话
分类会话（classify session）即分类规则，规则被添加到指定的分类表中。

#### 创建/删除分类规则
```
VLIB_CLI_COMMAND (classify_session_command, static) = {
    .path = "classify session",
    .short_help =
    "classify session [hit-next|l2-input-hit-next|l2-output-hit-next|"
    "acl-hit-next <next_index>|policer-hit-next <policer_name>]"
    "\n table-index <nn> match [hex] [l2] [l3 ip4] [opaque-index <index>]"
    "\n [action set-ip4-fib-id|set-ip6-fib-id|set-sr-policy-index <n>] [del]",
    .function = classify_session_command_fn,
};

static clib_error_t *
classify_session_command_fn (vlib_main_t * vm,
			     unformat_input_t * input,
			     vlib_cli_command_t * cmd)
{
  vnet_classify_main_t *cm = &vnet_classify_main;
  int is_add = 1;
  u32 table_index = ~0;
  u32 hit_next_index = ~0;
  u64 opaque_index = ~0;
  u8 *match = 0;
  i32 advance = 0;
  u32 action = 0;
  u32 metadata = 0;
  int i, rv;

  while (unformat_check_input (input) != UNFORMAT_END_OF_INPUT)
  {
      if (unformat (input, "del"))
	is_add = 0;
      else if (unformat (input, "hit-next %U", unformat_ip_next_index,
			 &hit_next_index))
	;
      else
	if (unformat
	    (input, "l2-input-hit-next %U", unformat_l2_input_next_index,
	     &hit_next_index))
	;
      else
	if (unformat
	    (input, "l2-output-hit-next %U", unformat_l2_output_next_index,
	     &hit_next_index))
	;
      else if (unformat (input, "acl-hit-next %U", unformat_acl_next_index,
			 &hit_next_index))
	;
      else if (unformat (input, "policer-hit-next %U",
			 unformat_policer_next_index, &hit_next_index))
	;
      else if (unformat (input, "opaque-index %lld", &opaque_index))
	;
      else if (unformat (input, "match %U", unformat_classify_match,
			 cm, &match, table_index))
	;
      else if (unformat (input, "advance %d", &advance))
	;
      else if (unformat (input, "table-index %d", &table_index))
	;
      else if (unformat (input, "action set-ip4-fib-id %d", &metadata))
	action = 1;
      else if (unformat (input, "action set-ip6-fib-id %d", &metadata))
	action = 2;
      else if (unformat (input, "action set-sr-policy-index %d", &metadata))
	action = 3;
      else
	{
	  /* Try registered opaque-index unformat fns */
	  for (i = 0; i < vec_len (cm->unformat_opaque_index_fns); i++)
	    {
	      if (unformat (input, "%U", cm->unformat_opaque_index_fns[i],
			    &opaque_index))
		goto found_opaque;
	    }
	  break;
	}
    found_opaque:
      ;
  }

  if (table_index == ~0)
    return clib_error_return (0, "Table index required");

  if (is_add && match == 0)
    return clib_error_return (0, "Match value required");

  rv = vnet_classify_add_del_session (cm, table_index, match,
				      hit_next_index,
				      opaque_index, advance,
				      action, metadata, is_add);

  switch (rv)
  {
    case 0:
      break;

    default:
      return clib_error_return (0,
				"vnet_classify_add_del_session returned %d",
				rv);
  }

  return 0;
}

int
vnet_classify_add_del_session (vnet_classify_main_t * cm,
			       u32 table_index,
			       u8 * match,
			       u32 hit_next_index,
			       u32 opaque_index,
			       i32 advance,
			       u8 action, u32 metadata, int is_add)
{
  vnet_classify_table_t *t;
  vnet_classify_entry_5_t _max_e __attribute__ ((aligned (16)));
  vnet_classify_entry_t *e;
  int i, rv;

  if (pool_is_free_index (cm->tables, table_index))
    return VNET_API_ERROR_NO_SUCH_TABLE;

  t = pool_elt_at_index (cm->tables, table_index);

  e = (vnet_classify_entry_t *) & _max_e;
  e->next_index = hit_next_index;
  e->opaque_index = opaque_index;
  e->advance = advance;
  e->hits = 0;
  e->last_heard = 0;
  e->flags = 0;
  e->action = action;
  if (e->action == CLASSIFY_ACTION_SET_IP4_FIB_INDEX)
    e->metadata = fib_table_find_or_create_and_lock (FIB_PROTOCOL_IP4,
						     metadata,
						     FIB_SOURCE_CLASSIFY);
  else if (e->action == CLASSIFY_ACTION_SET_IP6_FIB_INDEX)
    e->metadata = fib_table_find_or_create_and_lock (FIB_PROTOCOL_IP6,
						     metadata,
						     FIB_SOURCE_CLASSIFY);
  else if (e->action == CLASSIFY_ACTION_SET_METADATA)
    e->metadata = metadata;
  else
    e->metadata = 0;

  /* Copy key data, honoring skip_n_vectors */
  clib_memcpy_fast (&e->key, match + t->skip_n_vectors * sizeof (u32x4),
		    t->match_n_vectors * sizeof (u32x4));

  /* Clear don't-care bits; likely when dynamically creating sessions */
  for (i = 0; i < t->match_n_vectors; i++)
    e->key[i] &= t->mask[i];

  rv = vnet_classify_add_del (t, e, is_add);

  vnet_classify_entry_release_resource (e);

  if (rv)
    return VNET_API_ERROR_NO_SUCH_ENTRY;
  return 0;
}

int
vnet_classify_add_del (vnet_classify_table_t * t,
		       vnet_classify_entry_t * add_v, int is_add)
{
  u32 bucket_index;
  vnet_classify_bucket_t *b, tmp_b;
  vnet_classify_entry_t *v, *new_v, *save_new_v, *working_copy, *save_v;
  u32 value_index;
  int rv = 0;
  int i;
  u64 hash, new_hash;
  u32 limit;
  u32 old_log2_pages, new_log2_pages;
  u32 thread_index = vlib_get_thread_index ();
  u8 *key_minus_skip;
  int resplit_once = 0;
  int mark_bucket_linear;

  ASSERT ((add_v->flags & VNET_CLASSIFY_ENTRY_FREE) == 0);

  key_minus_skip = (u8 *) add_v->key;
  key_minus_skip -= t->skip_n_vectors * sizeof (u32x4);

  hash = vnet_classify_hash_packet (t, key_minus_skip);

  bucket_index = hash & (t->nbuckets - 1);
  b = &t->buckets[bucket_index];

  hash >>= t->log2_nbuckets;

  clib_spinlock_lock (&t->writer_lock);

  /* First elt in the bucket? */
  if (b->offset == 0)
  {
      if (is_add == 0)
	{
	  rv = -1;
	  goto unlock;
	}

      v = vnet_classify_entry_alloc (t, 0 /* new_log2_pages */ );
      clib_memcpy_fast (v, add_v, sizeof (vnet_classify_entry_t) +
			t->match_n_vectors * sizeof (u32x4));
      v->flags &= ~(VNET_CLASSIFY_ENTRY_FREE);
      vnet_classify_entry_claim_resource (v);

      tmp_b.as_u64 = 0;
      tmp_b.offset = vnet_classify_get_offset (t, v);

      b->as_u64 = tmp_b.as_u64;
      t->active_elements++;

      goto unlock;
  }

  make_working_copy (t, b);

  save_v = vnet_classify_get_entry (t, t->saved_bucket.offset);
  value_index = hash & ((1 << t->saved_bucket.log2_pages) - 1);
  limit = t->entries_per_page;
  if (PREDICT_FALSE (b->linear_search))
  {
      value_index = 0;
      limit *= (1 << b->log2_pages);
  }

  if (is_add)
  {
      /*
       * For obvious (in hindsight) reasons, see if we're supposed to
       * replace an existing key, then look for an empty slot.
       */

      for (i = 0; i < limit; i++)
	{
	  v = vnet_classify_entry_at_index (t, save_v, value_index + i);

	  if (!memcmp
	      (v->key, add_v->key, t->match_n_vectors * sizeof (u32x4)))
	    {
	      clib_memcpy_fast (v, add_v, sizeof (vnet_classify_entry_t) +
				t->match_n_vectors * sizeof (u32x4));
	      v->flags &= ~(VNET_CLASSIFY_ENTRY_FREE);
	      vnet_classify_entry_claim_resource (v);

	      CLIB_MEMORY_BARRIER ();
	      /* Restore the previous (k,v) pairs */
	      b->as_u64 = t->saved_bucket.as_u64;
	      goto unlock;
	    }
	}
      for (i = 0; i < limit; i++)
	{
	  v = vnet_classify_entry_at_index (t, save_v, value_index + i);

	  if (vnet_classify_entry_is_free (v))
	    {
	      clib_memcpy_fast (v, add_v, sizeof (vnet_classify_entry_t) +
				t->match_n_vectors * sizeof (u32x4));
	      v->flags &= ~(VNET_CLASSIFY_ENTRY_FREE);
	      vnet_classify_entry_claim_resource (v);

	      CLIB_MEMORY_BARRIER ();
	      b->as_u64 = t->saved_bucket.as_u64;
	      t->active_elements++;
	      goto unlock;
	    }
	}
      /* no room at the inn... split case... */
  }
  else
  {
      for (i = 0; i < limit; i++)
	{
	  v = vnet_classify_entry_at_index (t, save_v, value_index + i);

	  if (!memcmp
	      (v->key, add_v->key, t->match_n_vectors * sizeof (u32x4)))
	    {
	      vnet_classify_entry_release_resource (v);
	      clib_memset (v, 0xff, sizeof (vnet_classify_entry_t) +
			   t->match_n_vectors * sizeof (u32x4));
	      v->flags |= VNET_CLASSIFY_ENTRY_FREE;

	      CLIB_MEMORY_BARRIER ();
	      b->as_u64 = t->saved_bucket.as_u64;
	      t->active_elements--;
	      goto unlock;
	    }
	}
      rv = -3;
      b->as_u64 = t->saved_bucket.as_u64;
      goto unlock;
  }

  old_log2_pages = t->saved_bucket.log2_pages;
  new_log2_pages = old_log2_pages + 1;
  working_copy = t->working_copies[thread_index];

  if (t->saved_bucket.linear_search)
    goto linear_resplit;

  mark_bucket_linear = 0;

  new_v = split_and_rehash (t, working_copy, old_log2_pages, new_log2_pages);

  if (new_v == 0)
  {
    try_resplit:
      resplit_once = 1;
      new_log2_pages++;

      new_v = split_and_rehash (t, working_copy, old_log2_pages,
				new_log2_pages);
      if (new_v == 0)
	{
	mark_linear:
	  new_log2_pages--;

	linear_resplit:
	  /* pinned collisions, use linear search */
	  new_v = split_and_rehash_linear (t, working_copy, old_log2_pages,
					   new_log2_pages);
	  /* A new linear-search bucket? */
	  if (!t->saved_bucket.linear_search)
	    t->linear_buckets++;
	  mark_bucket_linear = 1;
	}
  }

  /* Try to add the new entry */
  save_new_v = new_v;

  key_minus_skip = (u8 *) add_v->key;
  key_minus_skip -= t->skip_n_vectors * sizeof (u32x4);

  new_hash = vnet_classify_hash_packet_inline (t, key_minus_skip);
  new_hash >>= t->log2_nbuckets;
  new_hash &= (1 << new_log2_pages) - 1;

  limit = t->entries_per_page;
  if (mark_bucket_linear)
  {
      limit *= (1 << new_log2_pages);
      new_hash = 0;
  }

  for (i = 0; i < limit; i++)
  {
      new_v = vnet_classify_entry_at_index (t, save_new_v, new_hash + i);

      if (vnet_classify_entry_is_free (new_v))
	{
	  clib_memcpy_fast (new_v, add_v, sizeof (vnet_classify_entry_t) +
			    t->match_n_vectors * sizeof (u32x4));
	  new_v->flags &= ~(VNET_CLASSIFY_ENTRY_FREE);
	  vnet_classify_entry_claim_resource (new_v);

	  goto expand_ok;
	}
  }
  /* Crap. Try again */
  vnet_classify_entry_free (t, save_new_v, new_log2_pages);

  if (resplit_once)
    goto mark_linear;
  else
    goto try_resplit;

expand_ok:
  tmp_b.log2_pages = new_log2_pages;
  tmp_b.offset = vnet_classify_get_offset (t, save_new_v);
  tmp_b.linear_search = mark_bucket_linear;

  CLIB_MEMORY_BARRIER ();
  b->as_u64 = tmp_b.as_u64;
  t->active_elements++;
  v = vnet_classify_get_entry (t, t->saved_bucket.offset);
  vnet_classify_entry_free (t, v, old_log2_pages);

unlock:
  clib_spinlock_unlock (&t->writer_lock);
  return rv;
}
```

### 在接口使能
### 在flow使能