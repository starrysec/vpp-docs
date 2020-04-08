## bihash

bihash(Bounded-index extensible hashing)：有界索引可扩展哈希，对于合适数量的哈希冲突，算法执行线程安全的恒定时间查找。
计算出的哈希值h(k)具有关于key空间的合理统计，对于所有k值，h(k)都不会等于0或1。

每个桶大小(bucket-array-size)为2的指数，桶中包含了“后续存储(backing store)”的索引(in a private vppinfra memory heap)和大小字段(log2_pages)，
对应的1、2、3、4...连续页面（page）中包含了key/value对。

当单个页面填满时，我们分配两个连续的页面，并使用附加位为每个key/value对重新计算h(k)将key/value对分配到“顶部”和“底部”页面。

在查找时，首先使用log2_nbuckets计算h(k)来选择桶，然后从桶中找到页面的起始位置，最后使用log2_pages从h(k)计算页面的偏移量，其中将包含要查找的key/value对。

* [bihash参考1](https://aijishu.com/a/1060000000017558)
* [bihash参考2](https://blog.csdn.net/xftony/article/details/80535717)

### 基本结构

```
/** template key/value backing page structure */
typedef struct clib_bihash_value
{
  union
  {

    clib_bihash_kv kvp[BIHASH_KVP_PER_PAGE]; /**< the actual key/value pairs, default value is 4 */
    clib_bihash_value *next_free;            /**< used when a KVP page (or block thereof) is on a freelist */
  };
} clib_bihash_value_t
```

```
/** bihash bucket structure */
  typedef struct
{
  union
  {
    struct
    {
      u32 offset;  /**< backing page offset in the clib memory heap */
      u8 pad[3];   /**< log2 (size of the packing page block) */
      u8 log2_pages;
    };
    u64 as_u64;
  };
} clib_bihash_bucket_t;
```

```
/** A bounded index extensible hash table */
typedef struct
{
  clib_bihash_bucket_t *buckets;  /**< Hash bucket vector, power-of-two in size */
  volatile u32 *writer_lock;      /**< Writer lock, in its own cache line */
  BVT (clib_bihash_value) ** working_copies;
                                  /**< Working copies (various sizes), to avoid locking against readers */
  clib_bihash_bucket_t saved_bucket; 
                                  /**< Saved bucket pointer */
  u32 nbuckets;			          /**< Number of hash buckets */
  u32 log2_nbuckets;		      /**< lg(nbuckets) */
  u8 *name;                       /**< hash table name */
  BVT (clib_bihash_value) ** freelists;
				                  /**< power of two freelist vector */
  uword alloc_arena;		      /**< memory allocation arena  */
  uword alloc_arena_next;	      /**< first available mem chunk */
  uword alloc_arena_size;	      /**< size of the arena */
} clib_bihash_t;
```

### 接口

```
/** Get pointer to value page given its clib mheap offset */
static inline void *clib_bihash_get_value (clib_bihash * h, uword offset);

/** Get clib mheap offset given a pointer */
static inline uword clib_bihash_get_offset (clib_bihash * h, void *v);

/** initialize a bounded index extensible hash table

    @param h - the bi-hash table to initialize
    @param name - name of the hash table
    @param nbuckets - the number of buckets, will be rounded up to
a power of two
    @param memory_size - clib mheap size, in bytes
*/

void clib_bihash_init
  (clib_bihash * h, char *name, u32 nbuckets, uword memory_size);

/** Destroy a bounded index extensible hash table
    @param h - the bi-hash table to free
*/

void clib_bihash_free (clib_bihash * h);

/** Add or delete a (key,value) pair from a bi-hash table

    @param h - the bi-hash table to search
    @param add_v - the (key,value) pair to add
    @param is_add - add=1, delete=0
    @returns 0 on success, < 0 on error
    @note This function will replace an existing (key,value) pair if the
    new key matches an existing key
*/
int clib_bihash_add_del (clib_bihash * h, clib_bihash_kv * add_v, int is_add);


/** Search a bi-hash table, use supplied hash code

    @param h - the bi-hash table to search
    @param hash - the hash code
    @param in_out_kv - (key,value) pair containing the search key
    @returns 0 on success (with in_out_kv set), < 0 on error
*/
int clib_bihash_search_inline_with_hash
  (clib_bihash * h, u64 hash, clib_bihash_kv * in_out_kv);

/** Search a bi-hash table

    @param h - the bi-hash table to search
    @param in_out_kv - (key,value) pair containing the search key
    @returns 0 on success (with in_out_kv set), < 0 on error
*/
int clib_bihash_search_inline (clib_bihash * h, clib_bihash_kv * in_out_kv);

/** Prefetch a bi-hash bucket given a hash code

    @param h - the bi-hash table to search
    @param hash - the hash code
    @note see also clib_bihash_hash to compute the code
*/
void clib_bihash_prefetch_bucket (clib_bihash * h, u64 hash);

/** Prefetch bi-hash (key,value) data given a hash code

    @param h - the bi-hash table to search
    @param hash - the hash code
    @note assumes that the bucket has been prefetched, see
     clib_bihash_prefetch_bucket
*/
void clib_bihash_prefetch_data (clib_bihash * h, u64 hash);

/** Search a bi-hash table

    @param h - the bi-hash table to search
    @param search_key - (key,value) pair containing the search key
    @param valuep - (key,value) set to search result
    @returns 0 on success (with valuep set), < 0 on error
    @note used in situations where key modification is not desired
*/
int clib_bihash_search_inline_2
  (clib_bihash * h, clib_bihash_kv * search_key, clib_bihash_kv * valuep);

/* Calback function for walking a bihash table
 *
 * @param kv - KV pair visited
 * @param ctx - Context passed to the walk
 * @return BIHASH_WALK_CONTINUE to continue BIHASH_WALK_STOP to stop
 */
typedef int (*clib_bihash_foreach_key_value_pair_cb) (clib_bihash_kv * kv,
						      void *ctx);

/** Visit active (key,value) pairs in a bi-hash table

    @param h - the bi-hash table to search
    @param callback - function to call with each active (key,value) pair
    @param arg - arbitrary second argument passed to the callback function
    First argument is the (key,value) pair to visit
*/
void clib_bihash_foreach_key_value_pair (clib_bihash * h,
					 clib_bihash_foreach_key_value_pair_cb
					 * callback, void *arg);
```