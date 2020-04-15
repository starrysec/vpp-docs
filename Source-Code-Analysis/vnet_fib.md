## FIB

### ip4_fib_t

** ipv4 fib**
```
typedef struct ip4_fib_t_
{
  /** Required for pool_get_aligned */
  CLIB_CACHE_LINE_ALIGN_MARK(cacheline0);

  /**
   * Mtrie for fast lookups. Hash is used to maintain overlapping prefixes.
   * First member so it's in the first cacheline.
   */
  ip4_fib_mtrie_t mtrie;

  /* Hash table for each prefix length mapping. */
  uword *fib_entry_by_dst_address[33];

  /* Table ID (hash key) for this FIB. */
  u32 table_id;

  /* Index into FIB vector. */
  u32 index;
} ip4_fib_t;
```
** 协议无关的fib table **
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

### create fib table

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

** 核心函数： **
```
static u32
ip4_create_fib_with_table_id (u32 table_id,
                              fib_source_t src)
{
    // fib table
    fib_table_t *fib_table;
    // ipv4 fib
    ip4_fib_t *v4_fib;
    void *old_heap;

    // 从fib table的vector中申请一个fib table
    pool_get(ip4_main.fibs, fib_table);
    clib_memset(fib_table, 0, sizeof(*fib_table));

    old_heap = clib_mem_set_heap (ip4_main.mtrie_mheap);
    pool_get_aligned(ip4_main.v4_fibs, v4_fib, CLIB_CACHE_LINE_BYTES);
    clib_mem_set_heap (old_heap);

    ASSERT((fib_table - ip4_main.fibs) ==
           (v4_fib - ip4_main.v4_fibs));

    // which protocol this table servers.
    fib_table->ft_proto = FIB_PROTOCOL_IP4;
    // 
    fib_table->ft_index = v4_fib->index = (fib_table - ip4_main.fibs);

    hash_set (ip4_main.fib_index_by_table_id, table_id, fib_table->ft_index);

    fib_table->ft_table_id =
    v4_fib->table_id =
        table_id;
    fib_table->ft_flow_hash_config = IP_FLOW_HASH_DEFAULT;
    
    fib_table_lock(fib_table->ft_index, FIB_PROTOCOL_IP4, src);

    ip4_mtrie_init(&v4_fib->mtrie);

    /*
     * add the special entries into the new FIB
     */
    int ii;

    for (ii = 0; ii < ARRAY_LEN(ip4_specials); ii++)
    {
    fib_prefix_t prefix = ip4_specials[ii].ift_prefix;

    prefix.fp_addr.ip4.data_u32 =
        clib_host_to_net_u32(prefix.fp_addr.ip4.data_u32);

    fib_table_entry_special_add(fib_table->ft_index,
                    &prefix,
                    ip4_specials[ii].ift_source,
                    ip4_specials[ii].ift_flag);
    }

    return (fib_table->ft_index);
}
```
