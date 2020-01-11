## 二进制API支持

VPP提供了一种二进制API方案，以允许各种各样的客户端代码对数据平面表进行编程。在撰写本文时，有数百种二进制API。

消息在*.api文件中定义。如今，大约有80个api文件，随着人们添加可编程功能，更多文件将陆续到来。API文件编译器源代码位于src/tools/vppapigen中。

在src/vnet/interface.api中，这是典型的请求/响应消息定义：
```
autoreply define sw_interface_set_flags
{
  u32 client_index;
  u32 context;
  u32 sw_if_index;
  /* 1 = up, 0 = down */
  u8 admin_up_down;
};
```

首先，API编译器将此定义生成到vpp/build-root/install-vpp_debug-native/vpp/include/vnet/ interface.api.h中，如下所示：
```
/****** Message ID / handler enum ******/

#ifdef vl_msg_id
vl_msg_id(VL_API_SW_INTERFACE_SET_FLAGS, vl_api_sw_interface_set_flags_t_handler)
vl_msg_id(VL_API_SW_INTERFACE_SET_FLAGS_REPLY, vl_api_sw_interface_set_flags_reply_t_handler)
#endif
/****** Message names ******/

#ifdef vl_msg_name
vl_msg_name(vl_api_sw_interface_set_flags_t, 1)
vl_msg_name(vl_api_sw_interface_set_flags_reply_t, 1)
#endif
/****** Message name, crc list ******/

#ifdef vl_msg_name_crc_list
#define foreach_vl_msg_name_crc_interface \
_(VL_API_SW_INTERFACE_SET_FLAGS, sw_interface_set_flags, f890584a) \
_(VL_API_SW_INTERFACE_SET_FLAGS_REPLY, sw_interface_set_flags_reply, dfbf3afa) \
#endif
/****** Typedefs *****/

#ifdef vl_typedefs
#ifndef defined_sw_interface_set_flags
#define defined_sw_interface_set_flags
typedef VL_API_PACKED(struct _vl_api_sw_interface_set_flags {
    u16 _vl_msg_id;
    u32 client_index;
    u32 context;
    u32 sw_if_index;
    u8 admin_up_down;
}) vl_api_sw_interface_set_flags_t;
#endif

#ifndef defined_sw_interface_set_flags_reply
#define defined_sw_interface_set_flags_reply
typedef VL_API_PACKED(struct _vl_api_sw_interface_set_flags_reply {
    u16 _vl_msg_id;
    u32 context;
    i32 retval;
}) vl_api_sw_interface_set_flags_reply_t;
#endif
...
#endif /* vl_typedefs */
```

要更改接口的管理状态，二进制api客户端会将vl_api_sw_interface_set_flags_t发送到VPP，VPP会以vl_api_sw_interface_set_flags_reply_t消息作为响应。

多层次的软件、传输类型和共享库实现了多种功能：

* API消息分配，跟踪，精美打印和重播。
* 通过全局共享内存，成对/私有共享内存和套接字的消息传输。
* 跨线程不安全的消息处理程序工作线程的屏障同步(Barrier Synchronization)。

正确的消息处理程序代码对用于向VPP传递消息或从VPP传递消息的传输一无所知。同时使用多种API消息传输类型相当简单。

由于历史原因，二进制api消息（假定）以网络字节顺序发送。在撰写本文时，我们正在认真考虑该选择是否合理。

### 消息分配

由于二进制API消息始终按顺序处理，因此我们尽可能使用环形分配器分配消息。与传统的内存分配器相比，该方案非常快，并且不会导致堆碎片。请参见src/vlibmemory/memory_shared.c中vl_msg_api_alloc_internal()。

无论采用哪种传输方式，二进制api消息始终跟随在一个msgbuf_t头之后：
```
/** Message header structure */
typedef struct msgbuf_
{
  svm_queue_t *q; /**< message allocated in this shmem ring  */
  u32 data_len;                  /**< message length not including header  */
  u32 gc_mark_timestamp;         /**< message garbage collector mark TS  */
  u8 data[0];                    /**< actual message begins here  */
} msgbuf_t;
```

这种结构使得跟踪消息变得非常容易，而无需解码-简单地保存data_len字节，并允许vl_msg_api_free() 来快速处理消息缓冲区。
```
void
vl_msg_api_free (void *a)
{
  msgbuf_t *rv;
  void *oldheap;
  api_main_t *am = &api_main;

  rv = (msgbuf_t *) (((u8 *) a) - offsetof (msgbuf_t, data));

  /*
   * Here's the beauty of the scheme.  Only one proc/thread has
   * control of a given message buffer. To free a buffer, we just clear the
   * queue field, and leave. No locks, no hits, no errors...
   */
  if (rv->q)
    {
      rv->q = 0;
      rv->gc_mark_timestamp = 0;
      <more code...>
      return;
    }
  <more code...>
}
```

### 消息跟踪和重放

有一点非常重要，就是VPP可以捕获和重放相当大的二进制API跟踪。涉及数十万个API事务的系统级问题可以在一秒钟或更短的时间内重新运行。部分重放允许对车轮掉落的点进行二进制搜索。可以在数据平面上添加脚手架，以在遇到复杂情况时触发。

通过二进制API跟踪，打印和重放，系统级错误以“在300,000个API事务之后，VPP数据平面停止转发流量，FIX IT！”这样的形式报告，然后可以离线去解决。

人们经常发现，控制平面客户端经过很长时间或在复杂环境下对数据平面进行了错误编程。没有直接证据，“这是一个数据平面问题！”

请参阅src/vlibmemory/memory_vlib::c vl_msg_api_process_file()和src/vlibapi/api_shared.c。另请参见调试CLI命令“api trace”。

### API跟踪重放警告

重放二进制API跟踪的vpp实例必须与捕获跟踪的vpp实例具有相同的message-ID编号空间。重放实例必须加载与捕获实例相同的插件集。否则，错误的API消息处理程序将处理API消息！

始终使用命令行参数（包括“api-trace on”节）启动vpp，因此vpp将从头开始跟踪二进制API消息：

```
api-trace {
  on
}
```

给定一个api跟踪到/tmp/api_trace下，请执行以下操作：
```
DBGvpp# api trace custom-dump /tmp/api_trace
vl_api_trace_plugin_msg_ids: abf_54307ba2 first 846 last 855
vl_api_trace_plugin_msg_ids: acl_0d7265b0 first 856 last 893
vl_api_trace_plugin_msg_ids: cdp_8f707b96 first 894 last 895
vl_api_trace_plugin_msg_ids: flowprobe_f2f0286c first 898 last 901
<etc>
```

在这里，我们看到了“abf”，“acl”，“cdp”和“flowprobe”插件。使用插件列表来构造匹配的“plugins”命令行参数节：
```
plugins {
    ## Disable all plugins, selectively enable specific plugins
    plugin default { disable }
    plugin abf_plugin.so { enable }
    plugin acl_plugin.so { enable }
    plugin cdp_plugin.so { enable }
    plugin flowprobe_plugin.so { enable }
}
```

首先，使用捕获跟踪的同一vpp映像重放它。重建vpp重放实例，添加脚手架以方便在复杂条件或类似条件下设置gdb断点，这是完全公平的。

### API跟踪接口问题

同样，可能需要制造[模拟]物理接口，以便API跟踪可以正确重放。跟踪源系统上的“显示界面”可以提供帮助。如上所示，API跟踪“custom-dump”可以很明显地创建出多少个环回接口。如果您看到正在创建和配置虚拟主机接口，则跟踪中的第一条此类配置消息将告诉您涉及了多少个物理接口。

```
SCRIPT: create_vhost_user_if socket /tmp/foosock server
SCRIPT: sw_interface_set_flags sw_if_index 3 admin-up
```

在这种情况下，可以很合理地猜测出需要创建两个环回接口来“帮助”正确重放跟踪。

通过在创建跟踪的系统上重放跟踪，可以在一定程度上缓解这些问题，但是在现场调试情况下这是不现实的。

### 客户端连接详情

通过C语言客户端跟VPP建立一个二进制API连接非常容易：
```
int
connect_to_vpe (char *client_name, int client_message_queue_length)
{
  vat_main_t *vam = &vat_main;
  api_main_t *am = &api_main;
  if (vl_client_connect_to_vlib ("/vpe-api", client_name,
                                client_message_queue_length) < 0)
    return -1;
  /* Memorize vpp's binary API message input queue address */
  vam->vl_input_queue = am->shmem_hdr->vl_input_queue;
  /* And our client index */
  vam->my_client_index = am->my_client_index;
  return 0;
}
```

32是client_message_queue_length的典型值。当需要向二进制API客户端发送API消息时，VPP无法阻止。VPP端的二进制API消息处理程序非常快。因此，在发送异步消息时，请务必热情地检查二进制API rx环！

**二进制API消息RX线程**

调用vl_client_connect_to_vlib会启动二进制API消息RX线程：
```
static void *
rx_thread_fn (void *arg)
{
  svm_queue_t *q;
  memory_client_main_t *mm = &memory_client_main;
  api_main_t *am = &api_main;
  int i;

  q = am->vl_input_queue;

  /* So we can make the rx thread terminate cleanly */
  if (setjmp (mm->rx_thread_jmpbuf) == 0)
    {
      mm->rx_thread_jmpbuf_valid = 1;
      /*
       * Find an unused slot in the per-cpu-mheaps array,
       * and grab it for this thread. We need to be able to
       * push/pop the thread heap without affecting other thread(s).
       */
      if (__os_thread_index == 0)
        {
          for (i = 0; i < ARRAY_LEN (clib_per_cpu_mheaps); i++)
            {
              if (clib_per_cpu_mheaps[i] == 0)
                {
                  /* Copy the main thread mheap pointer */
                  clib_per_cpu_mheaps[i] = clib_per_cpu_mheaps[0];
                  __os_thread_index = i;
                  break;
                }
            }
          ASSERT (__os_thread_index > 0);
        }
      while (1)
        vl_msg_api_queue_handler (q);
    }
  pthread_exit (0);
}
```

要自己处理二进制API消息队列，请使用vl_client_connect_to_vlib_no_rx_pthread。

**队列非空信号**

vl_msg_api_queue_handler(…)使用mutex/condvar信号唤醒，处理VPP->客户端流量，然后进入休眠状态。当VPP->客户端API消息队列从空过渡到非空时，VPP提供condvar广播。

VPP以很高的速度检查自己的二进制API输入队列。VPP根据数据平面数据包处理要求，以可变速率在“进程”上下文（也称为协作多任务线程上下文）中调用消息处理程序。

### 客户端断开连接详情

要与VPP断开连接，请调用vl_client_disconnect_from_vlib。如果客户端应用程序异常终止，请安排调用此函数。VPP竭尽全力为死掉的客户举行一场体面的葬礼，但VPP无法保证释放共享二进制API段中泄漏的内存。

### 发送二进制API消息到VPP



### 从VPP接收二进制API消息

#### 插件中API消息编号
