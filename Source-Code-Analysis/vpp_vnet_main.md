## vpp主程序

vpp主程序main函数位于`vpp/vnet/main.c`文件中。

### vpp main函数

#### vpp/vnet/main.c

```
int
main (int argc, char *argv[])
{
  int i;
  /* vlib_main_t记录着vpp全局信息 */
  vlib_main_t *vm = &vlib_global_main;
  /* api_main_t记录着api全局信息，声明设置message id的函数 */
  void vl_msg_api_set_first_available_msg_id (u16);
  /* 初始化主堆内存大小为1G */
  uword main_heap_size = (1ULL << 30);
  u8 *sizep;
  u32 size;
  int main_core = 1;
  cpu_set_t cpuset;
  void *main_heap;

#if __x86_64__
  CLIB_UNUSED (const char *msg)
    = "ERROR: This binary requires CPU with %s extensions.\n";
#define _(a,b)                                  \
    if (!clib_cpu_supports_ ## a ())            \
      {                                         \
	fprintf(stderr, msg, b);                \
	exit(1);                                \
      }

#if __AVX2__
  _(avx2, "AVX2")
#endif
#if __AVX__
    _(avx, "AVX")
#endif
#if __SSE4_2__
    _(sse42, "SSE4.2")
#endif
#if __SSE4_1__
    _(sse41, "SSE4.1")
#endif
#if __SSSE3__
    _(ssse3, "SSSE3")
#endif
#if __SSE3__
    _(sse3, "SSE3")
#endif
#undef _
#endif
    /*
     * Load startup config from file.
     * usage: vpp -c /etc/vpp/startup.conf
     */
    /* 以vpp -c /etc/vpp/startup.conf启动的vpp时，argc=3，解析参数 */
    if ((argc == 3) && !strncmp (argv[1], "-c", 2))
    {
      FILE *fp;
      char inbuf[4096];
      int argc_ = 1;
      char **argv_ = NULL;
      char *arg = NULL;
      char *p;
      
      /* 打开配置文件 */
      fp = fopen (argv[2], "r");
      if (fp == NULL)
	{
	  fprintf (stderr, "open configuration file '%s' failed\n", argv[2]);
	  return 1;
	}
      argv_ = calloc (1, sizeof (char *));
      if (argv_ == NULL)
	{
	  fclose (fp);
	  return 1;
	}
      arg = strndup (argv[0], 1024);
      if (arg == NULL)
	{
	  fclose (fp);
	  free (argv_);
	  return 1;
	}
      /* 按行存储配置信息，不包含注释行 */
      argv_[0] = arg;

      while (1)
	{
      /* 读取一行 */
	  if (fgets (inbuf, 4096, fp) == 0)
	    break;
	  p = strtok (inbuf, " \t\n");
	  while (p != NULL)
	    {
          /* 注释行 */
	      if (*p == '#')
		    break;
          /* 非注释行，保存到argv_中 */
	      argc_++;
	      char **tmp = realloc (argv_, argc_ * sizeof (char *));
	      if (tmp == NULL)
		    return 1;
	      argv_ = tmp;
	      arg = strndup (p, 1024);
	      if (arg == NULL)
		    return 1;
	      argv_[argc_ - 1] = arg;
	      p = strtok (NULL, " \t\n");
	    }
	}

      /* 关闭配置文件 */
      fclose (fp);

      char **tmp = realloc (argv_, (argc_ + 1) * sizeof (char *));
      if (tmp == NULL)
	return 1;
      argv_ = tmp;
      argv_[argc_] = NULL;

      argc = argc_;
      argv = argv_;
    }

  /*
   * Look for and parse the "heapsize" config parameter.
   * Manual since none of the clib infra has been bootstrapped yet.
   *
   * Format: heapsize <nn>[mM][gG]
   */

  /* 解析命令行其他参数 */
  for (i = 1; i < (argc - 1); i++)
    {
      /* 指定了plugin_path，则解析 */
      if (!strncmp (argv[i], "plugin_path", 11))
	{
	  if (i < (argc - 1))
	    vlib_plugin_path = argv[++i];
	}
      /* 指定了test_plugin_path，则解析 */
      if (!strncmp (argv[i], "test_plugin_path", 16))
	{
	  if (i < (argc - 1))
	    vat_plugin_path = argv[++i];
	}
      /* 指定了主堆内存大小，则解析 */
      else if (!strncmp (argv[i], "heapsize", 8))
	{
      /* 解析单位是mM还是gG */
	  sizep = (u8 *) argv[i + 1];
	  size = 0;
	  while (*sizep >= '0' && *sizep <= '9')
	    {
	      size *= 10;
	      size += *sizep++ - '0';
	    }
	  if (size == 0)
	    {
	      fprintf
		(stderr,
		 "warning: heapsize parse error '%s', use default %lld\n",
		 argv[i], (long long int) main_heap_size);
	      goto defaulted;
	    }
      /* 更新主堆内存大小 */
	  main_heap_size = size;

      /* 单位是gG，转换为字节 */
	  if (*sizep == 'g' || *sizep == 'G')
	    main_heap_size <<= 30;
      /* 单位是mM，转换为字节 */
	  else if (*sizep == 'm' || *sizep == 'M')
	    main_heap_size <<= 20;
	}
      /* 指定了主线程使用的逻辑核，则解析 */
      else if (!strncmp (argv[i], "main-core", 9))
	{
	  if (i < (argc - 1))
	    {
	      errno = 0;
          /* 解析核数，赋值给main_core */
	      unsigned long x = strtol (argv[++i], 0, 0);
	      if (errno == 0)
		    main_core = x;
	    }
	}
    }

defaulted:

  /* set process affinity for main thread */
  /* 设置主线程cpu核亲缘性，主线程绑定main_core */
  CPU_ZERO (&cpuset);
  CPU_SET (main_core, &cpuset);
  pthread_setaffinity_np (pthread_self (), sizeof (cpu_set_t), &cpuset);

  /* Set up the plugin message ID allocator right now... */
  /* api_main_t记录着api全局信息，设置开始message id */
  vl_msg_api_set_first_available_msg_id (VL_MSG_FIRST_AVAILABLE);

  /* 申请主堆内存 */
  if ((main_heap = clib_mem_init_thread_safe (0, main_heap_size)))
    {
      vlib_worker_thread_t tmp;

      /* 根据main_core查找执行主线程的numa id(cpu) ，由于numa系统可能存在多个物理cpu，每个cpu核心上存在多个逻辑核(超线程)情况，因此需求根据main_core查找 */
      vlib_get_thread_core_numa (&tmp, main_core);
      __os_numa_index = tmp.numa_id;

      /* 将申请的主堆作为numa id的cpu的numa堆内存使用 */
      clib_mem_set_per_numa_heap (main_heap);

      /* 创建hash表，记录调用的init函数列表 */
      vm->init_functions_called = hash_create (0, /* value bytes */ 0);
      /* 初始化环境 */
      vpe_main_init (vm);
      /* 进入vlib主函数 */
      return vlib_unix_main (argc, argv);
    }
  /* 申请主堆内存失败，退出 */
  else
    {
      {
	int rv __attribute__ ((unused)) =
	  write (2, "Main heap allocation failure!\r\n", 31);
      }
      return 1;
    }
}
```

```
static void
vpe_main_init (vlib_main_t * vm)
{
  /* 函数声明 */
  void vat_plugin_hash_create (void);

  /* 设置提示符 */
  if (CLIB_DEBUG > 0)
    vlib_unix_cli_set_prompt ("DBGvpp# ");
  else
    vlib_unix_cli_set_prompt ("vpp# ");

  /* Turn off network stack components which we don't want, SRP协议参见rfc2892 */
  vlib_mark_init_function_complete (vm, srp_init);

  /*
   * Create the binary api plugin hashes before loading plugins
   */
  /* 加载插件前，先创建用于记录接口引、函数、帮助的hash表 */
  vat_plugin_hash_create ();

  /* 未指定plugin_path时，自动从系统查找plugin_path */
  if (!vlib_plugin_path)
    vpp_find_plugin_path ();
}
```

### vlib main函数

#### vlib/unix/main.c

```
int
vlib_unix_main (int argc, char *argv[])
{
  vlib_main_t *vm = &vlib_global_main;	/* one and only time for this! */
  unformat_input_t input;
  clib_error_t *e;
  int i;

  vm->argv = (u8 **) argv;
  vm->name = argv[0];
  vm->heap_base = clib_mem_get_heap ();
  vm->heap_aligned_base = (void *)
    (((uword) vm->heap_base) & ~(VLIB_FRAME_ALIGN - 1));
  ASSERT (vm->heap_base);

  clib_time_init (&vm->clib_time);

  unformat_init_command_line (&input, (char **) vm->argv);
  if ((e = vlib_plugin_config (vm, &input)))
    {
      clib_error_report (e);
      return 1;
    }
  unformat_free (&input);

  i = vlib_plugin_early_init (vm);
  if (i)
    return i;

  unformat_init_command_line (&input, (char **) vm->argv);
  if (vm->init_functions_called == 0)
    vm->init_functions_called = hash_create (0, /* value bytes */ 0);
  e = vlib_call_all_config_functions (vm, &input, 1 /* early */ );
  if (e != 0)
    {
      clib_error_report (e);
      return 1;
    }
  unformat_free (&input);

  /* always load symbols, for signal handler and mheap memory get/put backtrace */
  clib_elf_main_init (vm->name);

  vec_validate (vlib_thread_stacks, 0);
  vlib_thread_stack_init (0);

  __os_thread_index = 0;
  vm->thread_index = 0;

  i = clib_calljmp (thread0, (uword) vm,
		    (void *) (vlib_thread_stacks[0] +
			      VLIB_THREAD_STACK_SIZE));
  return i;
}
```
