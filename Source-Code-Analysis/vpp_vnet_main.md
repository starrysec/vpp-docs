## vpp主程序

vpp主程序main函数位于`vpp/vnet/main.c`文件中，主要是启动主线程；工作线程根据绑核配置在plugins/dpdk/device/init.c中启动。

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
  /* 堆内存起始地址 */
  vm->heap_base = clib_mem_get_heap ();
  /* 堆内存对齐起始地址，方便申请vlib_frame_t */
  vm->heap_aligned_base = (void *)
    (((uword) vm->heap_base) & ~(VLIB_FRAME_ALIGN - 1));
  ASSERT (vm->heap_base);

  /* 初始化时间 */
  clib_time_init (&vm->clib_time);

  /* 读取plugins配置 */
  unformat_init_command_line (&input, (char **) vm->argv);
  if ((e = vlib_plugin_config (vm, &input)))
    {
      clib_error_report (e);
      return 1;
    }
  unformat_free (&input);

  /* 加载plugins并初始化 */
  i = vlib_plugin_early_init (vm);
  if (i)
    return i;

  unformat_init_command_line (&input, (char **) vm->argv);
  if (vm->init_functions_called == 0)
    vm->init_functions_called = hash_create (0, /* value bytes */ 0);
  /* 调用所有运行时节点初始化函数 */
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
  /* 给主线程申请栈内存 */
  vlib_thread_stack_init (0);

  __os_thread_index = 0;
  vm->thread_index = 0;

  /* 调到指定栈，执行主函数vlib_main */
  i = clib_calljmp (thread0, (uword) vm,
		    (void *) (vlib_thread_stacks[0] +
			      VLIB_THREAD_STACK_SIZE));
  return i;
}
```

#### vlib/main.c

```
/* Main function. */
int
vlib_main (vlib_main_t * volatile vm, unformat_input_t * input)
{
  clib_error_t *volatile error;
  vlib_node_main_t *nm = &vm->node_main;

  /* 控制平面api信号传递到queue的回调函数 */
  vm->queue_signal_callback = dummy_queue_signal_callback;

  /* 设置EventLogger环形队列大小为128K */
  if (!vm->elog_main.event_ring_size)
    vm->elog_main.event_ring_size = 128 << 10;
  /* 初始化EventLogger */
  elog_init (&vm->elog_main, vm->elog_main.event_ring_size);
  /* 打开Event logger */
  elog_enable_disable (&vm->elog_main, 1);
  /* 桩代码 */
  vl_api_set_elog_main (&vm->elog_main);
  /* 桩代码 */
  (void) vl_api_set_elog_trace_api_messages (1);

  /* Default name. Name for e.g. syslog. */
  if (!vm->name)
    vm->name = "VLIB";

  /* vlib物理内存初始化 */
  if ((error = vlib_physmem_init (vm)))
    {
      clib_error_report (error);
      goto done;
    }
  
  /*  */
  if ((error = vlib_map_stat_segment_init (vm)))
    {
      clib_error_report (error);
      goto done;
    }
  
  /*  */
  if ((error = vlib_buffer_main_init (vm)))
    {
      clib_error_report (error);
      goto done;
    }

  /*  */
  if ((error = vlib_thread_init (vm)))
    {
      clib_error_report (error);
      goto done;
    }

  /* Register static nodes so that init functions may use them. */
  vlib_register_all_static_nodes (vm);

  /* Set seed for random number generator.
     Allow user to specify seed to make random sequence deterministic. */
  if (!unformat (input, "seed %wd", &vm->random_seed))
    vm->random_seed = clib_cpu_time_now ();
  clib_random_buffer_init (&vm->random_buffer, vm->random_seed);

  /* Initialize node graph. */
  if ((error = vlib_node_main_init (vm)))
    {
      /* Arrange for graph hook up error to not be fatal when debugging. */
      if (CLIB_DEBUG > 0)
	clib_error_report (error);
      else
	goto done;
    }

  /* Direct call / weak reference, for vlib standalone use-cases */
  if ((error = vpe_api_init (vm)))
    {
      clib_error_report (error);
      goto done;
    }

  if ((error = vlibmemory_init (vm)))
    {
      clib_error_report (error);
      goto done;
    }

  if ((error = map_api_segment_init (vm)))
    {
      clib_error_report (error);
      goto done;
    }

  /* See unix/main.c; most likely already set up */
  if (vm->init_functions_called == 0)
    vm->init_functions_called = hash_create (0, /* value bytes */ 0);
  if ((error = vlib_call_all_init_functions (vm)))
    goto done;

  nm->timing_wheel = clib_mem_alloc_aligned (sizeof (TWT (tw_timer_wheel)),
					     CLIB_CACHE_LINE_BYTES);

  vec_validate (nm->data_from_advancing_timing_wheel, 10);
  _vec_len (nm->data_from_advancing_timing_wheel) = 0;

  /* Create the process timing wheel */
  TW (tw_timer_wheel_init) ((TWT (tw_timer_wheel) *) nm->timing_wheel,
			    0 /* no callback */ ,
			    10e-6 /* timer period 10us */ ,
			    ~0 /* max expirations per call */ );

  vec_validate (vm->pending_rpc_requests, 0);
  _vec_len (vm->pending_rpc_requests) = 0;
  vec_validate (vm->processing_rpc_requests, 0);
  _vec_len (vm->processing_rpc_requests) = 0;

  if ((error = vlib_call_all_config_functions (vm, input, 0 /* is_early */ )))
    goto done;

  /*
   * Use exponential smoothing, with a half-life of 1 second
   * reported_rate(t) = reported_rate(t-1) * K + rate(t)*(1-K)
   *
   * Sample every 20ms, aka 50 samples per second
   * K = exp (-1.0/20.0);
   * K = 0.95
   */
  vm->damping_constant = exp (-1.0 / 20.0);

  /* Sort per-thread init functions before we start threads */
  vlib_sort_init_exit_functions (&vm->worker_init_function_registrations);

  /* Call all main loop enter functions. */
  {
    clib_error_t *sub_error;
    sub_error = vlib_call_all_main_loop_enter_functions (vm);
    if (sub_error)
      clib_error_report (sub_error);
  }

  switch (clib_setjmp (&vm->main_loop_exit, VLIB_MAIN_LOOP_EXIT_NONE))
    {
    case VLIB_MAIN_LOOP_EXIT_NONE:
      vm->main_loop_exit_set = 1;
      break;

    case VLIB_MAIN_LOOP_EXIT_CLI:
      goto done;

    default:
      error = vm->main_loop_error;
      goto done;
    }

  /* 进入主线程循环 */
  vlib_main_loop (vm);

done:
  /* Call all exit functions. */
  {
    clib_error_t *sub_error;
    sub_error = vlib_call_all_main_loop_exit_functions (vm);
    if (sub_error)
      clib_error_report (sub_error);
  }

  if (error)
    clib_error_report (error);

  return 0;
}
```

```
static void
vlib_main_loop (vlib_main_t * vm)
{
  vlib_main_or_worker_loop (vm, /* is_main */ 1);
}
```

```
static_always_inline void
vlib_main_or_worker_loop (vlib_main_t * vm, int is_main)
{
  vlib_node_main_t *nm = &vm->node_main;
  vlib_thread_main_t *tm = vlib_get_thread_main ();
  uword i;
  u64 cpu_time_now;
  f64 now;
  vlib_frame_queue_main_t *fqm;
  u32 *last_node_runtime_indices = 0;
  u32 frame_queue_check_counter = 0;

  /* Initialize pending node vector. */
  if (is_main)
    {
      vec_resize (nm->pending_frames, 32);
      _vec_len (nm->pending_frames) = 0;
    }

  /* Mark time of main loop start. */
  if (is_main)
    {
      cpu_time_now = vm->clib_time.last_cpu_time;
      vm->cpu_time_main_loop_start = cpu_time_now;
    }
  else
    cpu_time_now = clib_cpu_time_now ();

  /* Pre-allocate interupt runtime indices and lock. */
  vec_alloc (nm->pending_interrupt_node_runtime_indices, 32);
  vec_alloc (last_node_runtime_indices, 32);
  if (!is_main)
    clib_spinlock_init (&nm->pending_interrupt_lock);

  /* Pre-allocate expired nodes. */
  if (!nm->polling_threshold_vector_length)
    nm->polling_threshold_vector_length = 10;
  if (!nm->interrupt_threshold_vector_length)
    nm->interrupt_threshold_vector_length = 5;

  vm->cpu_id = clib_get_current_cpu_id ();
  vm->numa_node = clib_get_current_numa_node ();
  os_set_numa_index (vm->numa_node);

  /* Start all processes. */
  if (is_main)
    {
      uword i;

      /*
       * Perform an initial barrier sync. Pays no attention to
       * the barrier sync hold-down timer scheme, which won't work
       * at this point in time.
       */
      vlib_worker_thread_initial_barrier_sync_and_release (vm);

      nm->current_process_index = ~0;
      for (i = 0; i < vec_len (nm->processes); i++)
	cpu_time_now = dispatch_process (vm, nm->processes[i], /* frame */ 0,
					 cpu_time_now);
    }

  while (1)
    {
      vlib_node_runtime_t *n;

      if (PREDICT_FALSE (_vec_len (vm->pending_rpc_requests) > 0))
	{
	  if (!is_main)
	    vl_api_send_pending_rpc_requests (vm);
	}

      if (!is_main)
	{
	  vlib_worker_thread_barrier_check ();
	  if (PREDICT_FALSE (vm->check_frame_queues +
			     frame_queue_check_counter))
	    {
	      u32 processed = 0;

	      if (vm->check_frame_queues)
		{
		  frame_queue_check_counter = 100;
		  vm->check_frame_queues = 0;
		}

	      vec_foreach (fqm, tm->frame_queue_mains)
		processed += vlib_frame_queue_dequeue (vm, fqm);

	      /* No handoff queue work found? */
	      if (processed)
		frame_queue_check_counter = 100;
	      else
		frame_queue_check_counter--;
	    }
	  if (PREDICT_FALSE (vec_len (vm->worker_thread_main_loop_callbacks)))
	    clib_call_callbacks (vm->worker_thread_main_loop_callbacks, vm);
	}

      /* Process pre-input nodes. */
      vec_foreach (n, nm->nodes_by_type[VLIB_NODE_TYPE_PRE_INPUT])
	cpu_time_now = dispatch_node (vm, n,
				      VLIB_NODE_TYPE_PRE_INPUT,
				      VLIB_NODE_STATE_POLLING,
				      /* frame */ 0,
				      cpu_time_now);

      /* Next process input nodes. */
      vec_foreach (n, nm->nodes_by_type[VLIB_NODE_TYPE_INPUT])
	cpu_time_now = dispatch_node (vm, n,
				      VLIB_NODE_TYPE_INPUT,
				      VLIB_NODE_STATE_POLLING,
				      /* frame */ 0,
				      cpu_time_now);

      if (PREDICT_TRUE (is_main && vm->queue_signal_pending == 0))
	vm->queue_signal_callback (vm);

      /* Next handle interrupts. */
      {
	/* unlocked read, for performance */
	uword l = _vec_len (nm->pending_interrupt_node_runtime_indices);
	uword i;
	if (PREDICT_FALSE (l > 0))
	  {
	    u32 *tmp;
	    if (!is_main)
	      {
		clib_spinlock_lock (&nm->pending_interrupt_lock);
		/* Re-read w/ lock held, in case another thread added an item */
		l = _vec_len (nm->pending_interrupt_node_runtime_indices);
	      }

	    tmp = nm->pending_interrupt_node_runtime_indices;
	    nm->pending_interrupt_node_runtime_indices =
	      last_node_runtime_indices;
	    last_node_runtime_indices = tmp;
	    _vec_len (last_node_runtime_indices) = 0;
	    if (!is_main)
	      clib_spinlock_unlock (&nm->pending_interrupt_lock);
	    for (i = 0; i < l; i++)
	      {
		n = vec_elt_at_index (nm->nodes_by_type[VLIB_NODE_TYPE_INPUT],
				      last_node_runtime_indices[i]);
		cpu_time_now =
		  dispatch_node (vm, n, VLIB_NODE_TYPE_INPUT,
				 VLIB_NODE_STATE_INTERRUPT,
				 /* frame */ 0,
				 cpu_time_now);
	      }
	  }
      }
      /* Input nodes may have added work to the pending vector.
         Process pending vector until there is nothing left.
         All pending vectors will be processed from input -> output. */
      for (i = 0; i < _vec_len (nm->pending_frames); i++)
	cpu_time_now = dispatch_pending_node (vm, i, cpu_time_now);
      /* Reset pending vector for next iteration. */
      _vec_len (nm->pending_frames) = 0;

      if (is_main)
	{
          /* *INDENT-OFF* */
          ELOG_TYPE_DECLARE (es) =
            {
              .format = "process tw start",
              .format_args = "",
            };
          ELOG_TYPE_DECLARE (ee) =
            {
              .format = "process tw end: %d",
              .format_args = "i4",
            };
          /* *INDENT-ON* */

	  struct
	  {
	    int nready_procs;
	  } *ed;

	  /* Check if process nodes have expired from timing wheel. */
	  ASSERT (nm->data_from_advancing_timing_wheel != 0);

	  if (PREDICT_FALSE (vm->elog_trace_graph_dispatch))
	    ed = ELOG_DATA (&vlib_global_main.elog_main, es);

	  nm->data_from_advancing_timing_wheel =
	    TW (tw_timer_expire_timers_vec)
	    ((TWT (tw_timer_wheel) *) nm->timing_wheel, vlib_time_now (vm),
	     nm->data_from_advancing_timing_wheel);

	  ASSERT (nm->data_from_advancing_timing_wheel != 0);

	  if (PREDICT_FALSE (vm->elog_trace_graph_dispatch))
	    {
	      ed = ELOG_DATA (&vlib_global_main.elog_main, ee);
	      ed->nready_procs =
		_vec_len (nm->data_from_advancing_timing_wheel);
	    }

	  if (PREDICT_FALSE
	      (_vec_len (nm->data_from_advancing_timing_wheel) > 0))
	    {
	      uword i;

	      for (i = 0; i < _vec_len (nm->data_from_advancing_timing_wheel);
		   i++)
		{
		  u32 d = nm->data_from_advancing_timing_wheel[i];
		  u32 di = vlib_timing_wheel_data_get_index (d);

		  if (vlib_timing_wheel_data_is_timed_event (d))
		    {
		      vlib_signal_timed_event_data_t *te =
			pool_elt_at_index (nm->signal_timed_event_data_pool,
					   di);
		      vlib_node_t *n =
			vlib_get_node (vm, te->process_node_index);
		      vlib_process_t *p =
			vec_elt (nm->processes, n->runtime_index);
		      void *data;
		      data =
			vlib_process_signal_event_helper (nm, n, p,
							  te->event_type_index,
							  te->n_data_elts,
							  te->n_data_elt_bytes);
		      if (te->n_data_bytes < sizeof (te->inline_event_data))
			clib_memcpy_fast (data, te->inline_event_data,
					  te->n_data_bytes);
		      else
			{
			  clib_memcpy_fast (data, te->event_data_as_vector,
					    te->n_data_bytes);
			  vec_free (te->event_data_as_vector);
			}
		      pool_put (nm->signal_timed_event_data_pool, te);
		    }
		  else
		    {
		      cpu_time_now = clib_cpu_time_now ();
		      cpu_time_now =
			dispatch_suspended_process (vm, di, cpu_time_now);
		    }
		}
	      _vec_len (nm->data_from_advancing_timing_wheel) = 0;
	    }
	}
      vlib_increment_main_loop_counter (vm);
      /* Record time stamp in case there are no enabled nodes and above
         calls do not update time stamp. */
      cpu_time_now = clib_cpu_time_now ();
      vm->loops_this_reporting_interval++;
      now = clib_time_now_internal (&vm->clib_time, cpu_time_now);
      /* Time to update loops_per_second? */
      if (PREDICT_FALSE (now >= vm->loop_interval_end))
	{
	  /* Next sample ends in 20ms */
	  if (vm->loop_interval_start)
	    {
	      f64 this_loops_per_second;

	      this_loops_per_second =
		((f64) vm->loops_this_reporting_interval) / (now -
							     vm->loop_interval_start);

	      vm->loops_per_second =
		vm->loops_per_second * vm->damping_constant +
		(1.0 - vm->damping_constant) * this_loops_per_second;
	      if (vm->loops_per_second != 0.0)
		vm->seconds_per_loop = 1.0 / vm->loops_per_second;
	      else
		vm->seconds_per_loop = 0.0;
	    }
	  /* New interval starts now, and ends in 20ms */
	  vm->loop_interval_start = now;
	  vm->loop_interval_end = now + 2e-4;
	  vm->loops_this_reporting_interval = 0;
	}
    }
}
```