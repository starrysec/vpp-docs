## FIB

在`vnet/ip/ip4.h`的ip4_main_t结构体定义了用于转发的MTries和全量FIBs。
其中表示FIB的结构体是fib_table_t，与协议无关。
表示转发的结构体是ip4_fib_t，与协议相关。

/**
 * @brief IPv4 main type.
 *
 * State of IPv4 VPP processing including:
 * - FIBs
 * - Feature indices used in feature topological sort
 * - Feature node run time references
 */

typedef struct ip4_main_t
{
  ip_lookup_main_t lookup_main;

  /** Vector of FIBs. */
  struct fib_table_t_ *fibs;

  /** Vector of MTries. */
  struct ip4_fib_t_ *v4_fibs;
  
  // 其他成员省略
} ip4_main_t;

### fib_table_t

```
/**
 * @brief 
 *   A protocol Independent FIB table
 */
typedef struct fib_table_t_
{
    /**
     * Which protocol this table serves. Used to switch on the union above.
     */
    fib_protocol_t ft_proto;

    /**
     * Table flags
     */
    fib_table_flags_t ft_flags;

    /**
     * per-source number of locks on the table
     */
    u32 *ft_locks;
    u32 ft_total_locks;

    /**
     * Table ID (hash key) for this FIB.
     */
    u32 ft_table_id;

    /**
     * Index into FIB vector.
     */
    fib_node_index_t ft_index;

    /**
     * flow hash configuration
     */
    u32 ft_flow_hash_config;

    /**
     * Per-source route counters
     */
    u32 *ft_src_route_counts;

    /**
     * Total route counters
     */
    u32 ft_total_route_counts;

    /**
     * Epoch - number of resyncs performed
     */
    u32 ft_epoch;

    /**
     * Table description
     */
    u8* ft_desc;
} fib_table_t;
```

### ip4_fib_t

```
/* The ‘non-forwarding’ ip4_fib_t contains all the entries in the table and, 
 * the ‘forwarding’ contains the entries that are matched against in the data-plane. 
 * The difference between the two sets are the entries that should not be matched in the data-plane. 
 * Each ip4_fib_t comprises an mtrie (for fast lookup in the data-plane) and a hash table per-prefix length (for lookup in the control plane). 
 * 
 * IPv6 also has the concept of forwarding and non-forwarding entries, 
 * however for IPv6 all the forwardind entries are stored in a single hash table (same goes for the non-forwarding). 
 * The key to the hash table includes the IPv6 table-id.
 */
typedef struct ip4_fib_t_
{
  /** Required for pool_get_aligned */
  CLIB_CACHE_LINE_ALIGN_MARK(cacheline0);

  /**
   * Mtrie for fast lookups. Hash is used to maintain overlapping prefixes.
   * First member so it's in the first cacheline.
   */
  // 用于转发的mtrie树
  ip4_fib_mtrie_t mtrie;

  /* Hash table for each prefix length mapping. */
  // 前缀长度->fib_table_t
  uword *fib_entry_by_dst_address[33];

  /* Table ID (hash key) for this FIB. */
  u32 table_id;

  /* Index into FIB vector. */
  u32 index;
} ip4_fib_t;
```

### 创建ipv4 fib table

```
/**
 * @brief Get or create an IPv4 fib.
 *
 * Get or create an IPv4 fib with the provided table ID.
 *
 * @param table_id
 *      When set to \c ~0, an arbitrary and unused fib ID is picked
 *      and can be retrieved with \c ret->table_id.
 *      Otherwise, the fib ID to be used to retrieve or create the desired fib.
 * @returns A pointer to the retrieved or created fib.
 *
 */
extern u32 ip4_fib_table_find_or_create_and_lock(u32 table_id,
                                                 fib_source_t src);
extern u32 ip4_fib_table_create_and_lock(fib_source_t src);
```

**核心函数**
```
static u32
ip4_create_fib_with_table_id (u32 table_id,
                              fib_source_t src)
{
    // FIB路由表
    fib_table_t *fib_table;
    // 用于转发的表
    ip4_fib_t *v4_fib;
    void *old_heap;

    // 从fib池中申请一个fib table
    pool_get(ip4_main.fibs, fib_table);
    clib_memset(fib_table, 0, sizeof(*fib_table));

    old_heap = clib_mem_set_heap (ip4_main.mtrie_mheap);
    pool_get_aligned(ip4_main.v4_fibs, v4_fib, CLIB_CACHE_LINE_BYTES);
    clib_mem_set_heap (old_heap);

    ASSERT((fib_table - ip4_main.fibs) == (v4_fib - ip4_main.v4_fibs));

    // which protocol this table servers.
    fib_table->ft_proto = FIB_PROTOCOL_IP4;
    // fib table index
    fib_table->ft_index = v4_fib->index = (fib_table - ip4_main.fibs);

    // Hash table mapping table id to fib index. ID space is not necessarily dense; index space is dense.
    // 设置key/value对：[table_id] -> [fib_table->ft_index]
    hash_set (ip4_main.fib_index_by_table_id, table_id, fib_table->ft_index);
    // 统一赋table id
    fib_table->ft_table_id = v4_fib->table_id = table_id;
    // 配置流hash计算方法，默认为5元组
    fib_table->ft_flow_hash_config = IP_FLOW_HASH_DEFAULT;
    
    // per-source number of locks and totoal number of locks on the table
    fib_table_lock(fib_table->ft_index, FIB_PROTOCOL_IP4, src);

    // 初始化用于转发的mtrie树
    ip4_mtrie_init(&v4_fib->mtrie);

    /*
     * add the special entries into the new FIB
     */
    int ii;

    // 向fib table中添加一些特殊前缀的路由，0.0.0.0/0，0.0.0.0/32，240.0.0.0/4，224.0.0.0/4，255.255.255.255/32
    for (ii = 0; ii < ARRAY_LEN(ip4_specials); ii++)
    {
      // 前缀
      fib_prefix_t prefix = ip4_specials[ii].ift_prefix;
      
      // 前缀地址赋值
      prefix.fp_addr.ip4.data_u32 = clib_host_to_net_u32(prefix.fp_addr.ip4.data_u32);

      // 路由插入到fib table中
      fib_table_entry_special_add(fib_table->ft_index, &prefix, ip4_specials[ii].ift_source, ip4_specials[ii].ift_flag);
    }

    // 返回fib index
    return (fib_table->ft_index);
}
```

### 销毁ipv4 fib table

```
void
ip4_fib_table_destroy (u32 fib_index)
{
    // 根据fib index获取fib table
    fib_table_t *fib_table = pool_elt_at_index(ip4_main.fibs, fib_index);
    // 根据fib index获取用于转发的mtries
    ip4_fib_t *v4_fib = pool_elt_at_index(ip4_main.v4_fibs, fib_index);
    u32 *n_locks;
    int ii;

    /*
     * remove all the specials we added when the table was created.
     * In reverse order so the default route is last.
     */
    for (ii = ARRAY_LEN(ip4_specials) - 1; ii >= 0; ii--)
    {
	  fib_prefix_t prefix = ip4_specials[ii].ift_prefix;

	  prefix.fp_addr.ip4.data_u32 =
	    clib_host_to_net_u32(prefix.fp_addr.ip4.data_u32);

      // 从fib table中删除路由
	  fib_table_entry_special_remove(fib_table->ft_index,
				       &prefix,
				       ip4_specials[ii].ift_source);
    }

    /*
     * validate no more routes.
     */
#ifdef CLIB_DEBUG
    if (0 != fib_table->ft_total_route_counts)
        fib_table_assert_empty(fib_table);
#endif

    vec_foreach(n_locks, fib_table->ft_src_route_counts)
    {
	  ASSERT(0 == *n_locks);
    }

    if (~0 != fib_table->ft_table_id)
    {
      // 移除key/value对：[table_id] -> [fib_table->ft_index]
	  hash_unset (ip4_main.fib_index_by_table_id, fib_table->ft_table_id);
    }

    vec_free(fib_table->ft_src_route_counts);
    // 释放转发mtrie的资源
    ip4_mtrie_free(&v4_fib->mtrie);

    // 删除fib和转发mtrie
    pool_put(ip4_main.v4_fibs, v4_fib);
    pool_put(ip4_main.fibs, fib_table);
}
```

### 在ipv4 fib table中查找

```
/*
 * ip4_fib_table_lookup
 *
 * Longest prefix match
 */
fib_node_index_t
ip4_fib_table_lookup (const ip4_fib_t *fib,
		      const ip4_address_t *addr,
		      u32 len)
{
    uword * hash, * result;
    i32 mask_len;
    u32 key;

    // 从转发mtries中查找
    for (mask_len = len; mask_len >= 0; mask_len--)
    {
      // 根据mask len获取hash表
	  hash = fib->fib_entry_by_dst_address[mask_len];
      // 构造在hash中查询的key（有地址和掩码构成）
	  key = (addr->data_u32 & ip4_main.fib_masks[mask_len]);

      // 在hash表中查找
	  result = hash_get (hash, key);

	  if (NULL != result) {
	    return (result[0]);
	  }
    }
    return (FIB_NODE_INDEX_INVALID);
}
```

### 插入ipv4 fib entry

```
void
ip4_fib_table_entry_insert (ip4_fib_t *fib,
			    const ip4_address_t *addr,
			    u32 len,
			    fib_node_index_t fib_entry_index)
{
    uword * hash, * result;
    u32 key;

    // 由地址和掩码构成的key
    key = (addr->data_u32 & ip4_main.fib_masks[len]);
    // 根据前缀长度，获取hash表
    hash = fib->fib_entry_by_dst_address[len];
    result = hash_get (hash, key);

    // 未找到，新建entry
    if (NULL == result) {
	  /*
	   * adding a new entry
	   */
      uword *old_heap;
      old_heap = clib_mem_set_heap (ip4_main.mtrie_mheap);

	  if (NULL == hash) {
	    hash = hash_create (32 /* elts */, sizeof (uword));
	    hash_set_flags (hash, HASH_FLAG_NO_AUTO_SHRINK);
	  }
      // 把key和entry对放到hash表中
	  hash = hash_set(hash, key, fib_entry_index);
	  fib->fib_entry_by_dst_address[len] = hash;
      clib_mem_set_heap (old_heap);
    }
    else
    {
	  ASSERT(0);
    }
}
```

### 删除ipv4 fib entry

```
void
ip4_fib_table_entry_remove (ip4_fib_t *fib,
			    const ip4_address_t *addr,
			    u32 len)
{
    uword * hash, * result;
    u32 key;

    key = (addr->data_u32 & ip4_main.fib_masks[len]);
    hash = fib->fib_entry_by_dst_address[len];
    result = hash_get (hash, key);

    if (NULL == result)
    {
	/*
	 * removing a non-existent entry. i'll allow it.
	 */
    }
    else 
    {
      uword *old_heap;

      old_heap = clib_mem_set_heap (ip4_main.mtrie_mheap);
      // hash表中删除
	  hash_unset(hash, key);
      clib_mem_set_heap (old_heap);
    }

    // 感觉是多余的，why???
    fib->fib_entry_by_dst_address[len] = hash;
}
```