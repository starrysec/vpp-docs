## bihash

bihash(Bounded-index extensible hashing)：有界索引可扩展哈希，对于合适数量的哈希冲突，算法执行线程安全的恒定时间查找。
计算出的哈希值h(k)具有关于key空间的合理统计，对于所有k值，h(k)都不会等于0或1。

每个桶大小(bucket-array-size)为2的指数，桶中包含了“后续存储(backing store)”的索引(in a private vppinfra memory heap)和大小字段(log2_pages)，
对应的1、2、3、4...连续页面（page）中包含了key/value对。

当单个页面填满时，我们分配两个连续的页面，并使用附加位为每个key/value对重新计算h(k)将key/value对分配到“顶部”和“底部”页面。

在查找时，首先使用lg(bucket-array-size)计算h(k)来选择桶，然后从桶中找到页面的起始位置，最后使用一个额外的log2_pages从h(k)计算页面的偏移量，其中将包含要查找的key/value对。

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
  u8 *name;			              /**< hash table name */
    BVT (clib_bihash_value) ** freelists;
				                  /**< power of two freelist vector */
  uword alloc_arena;		      /**< memory allocation arena  */
  uword alloc_arena_next;	      /**< first available mem chunk */
  uword alloc_arena_size;	      /**< size of the arena */
} clib_bihash_t;
```