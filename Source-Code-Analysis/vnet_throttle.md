## throttle

throttle即节流器，是提供给数据面用来决定给定的哈希是否应该被节流（随机节流）的一种机制。

### 节流器初始化

```
typedef struct throttle_t_
{
  // 时间间隔
  f64 time;
  // 为每个线程创建一个bitmap，每个bitmap最大存储64个节流器
  uword **bitmaps;
  // 为每个线程创意个随机种子
  u64 *seeds;
  // 每个线程保存一个上次种子修改时间
  f64 *last_seed_change_time;
} throttle_t;
```

```
void
throttle_init (throttle_t * t, u32 n_threads, f64 time)
{
  u32 i;

  // 初始化时间间隔，一般为0.001秒（1e-3）
  t->time = time;
  // 初始化bitmaps，为每个线程创建一个bitmap，每个bitmap最大存储64（512/8）个节流器位
  vec_validate (t->bitmaps, n_threads);
  // 初始化种子，为每个线程创建一个随机种子
  vec_validate (t->seeds, n_threads);
  // 初始化种子上次修改时间
  vec_validate (t->last_seed_change_time, n_threads);

  for (i = 0; i < n_threads; i++)
    vec_validate (t->bitmaps[i], (THROTTLE_BITS / BITS (uword)) - 1);
}
```

### 更新随机种子

```
always_inline u64
throttle_seed (throttle_t * t, u32 thread_index, f64 time_now)
{
  // 检查是否需要更新种子（间隔是否大于初始化设置的时间间隔）
  if (time_now - t->last_seed_change_time[thread_index] > t->time)
    {
      // 更新线程随机种子
      (void) random_u64 (&t->seeds[thread_index]);
      // 清空线程节流器位
      clib_memset (t->bitmaps[thread_index], 0, THROTTLE_BITS / BITS (u8));
      
      t->last_seed_change_time[thread_index] = time_now;
    }
  return t->seeds[thread_index];
}
```

### 节流检测（随机节流）

```
always_inline int
throttle_check (throttle_t * t, u32 thread_index, u64 hash, u64 seed)
{
  int drop;
  uword m;
  u32 w;

  // 在arp中，hash由构造arp的目的ip和出接口组成，并和seed异或使之随机化
  hash = clib_xxhash (hash ^ seed);

  /* Select bit number */
  // 根据随机化后的hash选择节流器位w
  hash &= THROTTLE_BITS - 1; // 将hash限定到0-511范围内
  w = hash / BITS (uword); // hash限定到0-31范围内
  // 获取节流器位
  m = (uword) 1 << (hash % BITS (uword)); // 1 << 0 ~ 1 << 31

  // 比较节流器位，确定是否节流（丢弃）
  drop = (t->bitmaps[thread_index][w] & m) != 0;
  // 设置节流器位
  t->bitmaps[thread_index][w] |= m;

  // 返回是否该被节流
  return (drop);
}
```