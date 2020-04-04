## Packet Generator

Packet Generator用法，请参见[官方文档](https://wiki.fd.io/view/VPP/How_To_Use_The_Packet_Generator_and_Packet_Tracer)。
* [Create/Delete Stream]()
* [Enable/Disable Stream]()
* [Packet-Generator Create/Delete]()
* [Packet-Generator Capture]()
* [Packet-Generator Configure]()
* [Show Packet-Generator]()

### 配置

#### vnet/pg/cli.c

**create/delete stream**
```
VLIB_CLI_COMMAND (new_stream_cli, static) = {
  .path = "packet-generator new",
  .function = new_stream,
  .short_help = "Create packet generator stream",
  .long_help =
  "Create packet generator stream\n"
  "\n"
  "Arguments:\n"
  "\n"
  "name STRING          sets stream name\n"
  "interface STRING     interface for stream output \n"
  "node NODE-NAME       node for stream output\n"
  "data STRING          specifies packet data\n"
  "pcap FILENAME        read packet data from pcap file\n"
  "rate PPS             rate to transfer packet data\n"
  "maxframe NPKTS       maximum number of packets per frame\n",
};

VLIB_CLI_COMMAND (del_stream_cli, static) = {
  .path = "packet-generator delete",
  .function = del_stream,
  .short_help = "Delete stream with given name",
};

static clib_error_t *
new_stream (vlib_main_t * vm,
        unformat_input_t * input, vlib_cli_command_t * cmd)
{
  clib_error_t *error = 0;
  u8 *tmp = 0;
  u32 maxframe, hw_if_index;
  unformat_input_t sub_input = { 0 };
  int sub_input_given = 0;
  vnet_main_t *vnm = vnet_get_main ();
  pg_main_t *pg = &pg_main;
  pg_stream_t s = { 0 };
  char *pcap_file_name;

  s.sw_if_index[VLIB_RX] = s.sw_if_index[VLIB_TX] = ~0;
  s.node_index = ~0;
  s.max_packet_bytes = s.min_packet_bytes = 64;
  s.buffer_bytes = vlib_buffer_get_default_data_size (vm);
  s.if_id = 0;
  s.n_max_frame = VLIB_FRAME_SIZE;
  pcap_file_name = 0;

  while (unformat_check_input (input) != UNFORMAT_END_OF_INPUT)
    {
      if (unformat (input, "name %v", &tmp))
    {
      if (s.name)
        vec_free (s.name);
      s.name = tmp;
    }

      else if (unformat (input, "node %U",
             unformat_vnet_hw_interface, vnm, &hw_if_index))
    {
      vnet_hw_interface_t *hi = vnet_get_hw_interface (vnm, hw_if_index);

      s.node_index = hi->output_node_index;
      s.sw_if_index[VLIB_TX] = hi->sw_if_index;
    }

      else if (unformat (input, "source pg%u", &s.if_id))
    ;

      else if (unformat (input, "node %U",
             unformat_vlib_node, vm, &s.node_index))
    ;
      else if (unformat (input, "maxframe %u", &maxframe))
    s.n_max_frame = s.n_max_frame < maxframe ? s.n_max_frame : maxframe;
      else if (unformat (input, "worker %u", &s.worker_index))
    ;

      else if (unformat (input, "interface %U",
             unformat_vnet_sw_interface, vnm,
             &s.sw_if_index[VLIB_RX]))
    ;
      else if (unformat (input, "tx-interface %U",
             unformat_vnet_sw_interface, vnm,
             &s.sw_if_index[VLIB_TX]))
    ;

      else if (unformat (input, "pcap %s", &pcap_file_name))
    ;

      else if (!sub_input_given
           && unformat (input, "data %U", unformat_input, &sub_input))
    sub_input_given++;

      else if (unformat_user (input, unformat_pg_stream_parameter, &s))
    ;

      else
    {
      error = clib_error_create ("unknown input `%U'",
                     format_unformat_error, input);
      goto done;
    }
    }

  if (!sub_input_given && !pcap_file_name)
    {
      error = clib_error_create ("no packet data given");
      goto done;
    }

  if (s.node_index == ~0)
    {
      if (pcap_file_name != 0)
    {
      vlib_node_t *n =
        vlib_get_node_by_name (vm, (u8 *) "ethernet-input");
      s.node_index = n->index;
    }
      else
    {
      error = clib_error_create ("output interface or node not given");
      goto done;
    }
    }

  {
    pg_node_t *n;

    if (s.node_index < vec_len (pg->nodes))
      n = pg->nodes + s.node_index;
    else
      n = 0;

    if (s.worker_index >= vlib_num_workers ())
      s.worker_index = 0;

    if (pcap_file_name != 0)
      {
    error = pg_pcap_read (&s, pcap_file_name);
    if (error)
      goto done;
    vec_free (pcap_file_name);
      }

    else if (n && n->unformat_edit
         && unformat_user (&sub_input, n->unformat_edit, &s))
      ;

    else if (!unformat_user (&sub_input, unformat_pg_payload, &s))
      {
    error = clib_error_create
      ("failed to parse packet data from `%U'",
       format_unformat_error, &sub_input);
    goto done;
      }
  }

  error = validate_stream (&s);
  if (error)
    return error;

  pg_stream_add (pg, &s);
  return 0;

done:
  pg_stream_free (&s);
  unformat_free (&sub_input);
  return error;
}

static clib_error_t *
del_stream (vlib_main_t * vm,
        unformat_input_t * input, vlib_cli_command_t * cmd)
{
  pg_main_t *pg = &pg_main;
  u32 i;

  if (!unformat (input, "%U",
         &unformat_hash_vec_string, pg->stream_index_by_name, &i))
    return clib_error_create ("expected stream name `%U'",
                  format_unformat_error, input);

  pg_stream_del (pg, i);
  return 0;
}
```

**enable/disable stream**
```
VLIB_CLI_COMMAND (enable_streams_cli, static) = {
  .path = "packet-generator enable-stream",
  .short_help = "Enable packet generator streams",
  .function = enable_disable_stream,
  .function_arg = 1,        /* is_enable */
};

VLIB_CLI_COMMAND (disable_streams_cli, static) = {
  .path = "packet-generator disable-stream",
  .short_help = "Disable packet generator streams",
  .function = enable_disable_stream,
  .function_arg = 0,        /* is_enable */
};

static clib_error_t *
enable_disable_stream (vlib_main_t * vm,
               unformat_input_t * input, vlib_cli_command_t * cmd)
{
  unformat_input_t _line_input, *line_input = &_line_input;
  pg_main_t *pg = &pg_main;
  int is_enable = cmd->function_arg != 0;
  u32 stream_index = ~0;

  if (!unformat_user (input, unformat_line_input, line_input))
    goto doit;

  while (unformat_check_input (line_input) != UNFORMAT_END_OF_INPUT)
  {
    // parse stream id by stream name.
    if (unformat (line_input, "%U", unformat_hash_vec_string,
            pg->stream_index_by_name, &stream_index));
    // error occured.
    else
      return clib_error_create ("unknown input `%U'",
                  format_unformat_error, line_input);
  }
  unformat_free (line_input);

doit:
  // enable or disable specified stream.
  pg_enable_disable (stream_index, is_enable);

  return 0;
}

void
pg_enable_disable (u32 stream_index, int is_enable)
{
  pg_main_t *pg = &pg_main;
  pg_stream_t *s;

  if (stream_index == ~0)
    {
      /* No stream specified: enable/disable all streams. */
      /* *INDENT-OFF* */
        pool_foreach (s, pg->streams, ({
            pg_stream_enable_disable (pg, s, is_enable);
        }));
    /* *INDENT-ON* */
    }
  else
    {
      /* enable/disable specified stream. */
      s = pool_elt_at_index (pg->streams, stream_index);
      pg_stream_enable_disable (pg, s, is_enable);
    }
}

/* Mark stream active or inactive. */
void
pg_stream_enable_disable (pg_main_t * pg, pg_stream_t * s, int want_enabled)
{
  vlib_main_t *vm;
  vnet_main_t *vnm = vnet_get_main ();
  pg_interface_t *pi = pool_elt_at_index (pg->interfaces, s->pg_if_index);

  want_enabled = want_enabled != 0;

  if (pg_stream_is_enabled (s) == want_enabled)
    /* No change necessary. */
    return;

  if (want_enabled)
    s->n_packets_generated = 0;

  /* Toggle enabled flag. */
  s->flags ^= PG_STREAM_FLAGS_IS_ENABLED;

  ASSERT (!pool_is_free (pg->streams, s));

  vec_validate (pg->enabled_streams, s->worker_index);
  pg->enabled_streams[s->worker_index] =
    clib_bitmap_set (pg->enabled_streams[s->worker_index], s - pg->streams,
             want_enabled);

  if (want_enabled)
    {
      vnet_hw_interface_set_flags (vnm, pi->hw_if_index,
                   VNET_HW_INTERFACE_FLAG_LINK_UP);

      vnet_sw_interface_set_flags (vnm, pi->sw_if_index,
                   VNET_SW_INTERFACE_FLAG_ADMIN_UP);
    }

  if (vlib_num_workers ())
    vm = vlib_get_worker_vlib_main (s->worker_index);
  else
    vm = vlib_get_main ();

  vlib_node_set_state (vm, pg_input_node.index,
               (clib_bitmap_is_zero
            (pg->enabled_streams[s->worker_index]) ?
            VLIB_NODE_STATE_DISABLED : VLIB_NODE_STATE_POLLING));

  s->packet_accumulator = 0;
  s->time_last_generate = 0;
}
```

**create packet generator**
```
VLIB_CLI_COMMAND (create_pg_if_cmd, static) = {
  .path = "create packet-generator",
  .short_help = "create packet-generator interface <interface name> [gso-enabled gso-size <size>]",
  .function = create_pg_if_cmd_fn,
};

static clib_error_t *
create_pg_if_cmd_fn (vlib_main_t * vm,
             unformat_input_t * input, vlib_cli_command_t * cmd)
{
  pg_main_t *pg = &pg_main;
  unformat_input_t _line_input, *line_input = &_line_input;
  u32 if_id, gso_enabled = 0, gso_size = 0;
  clib_error_t *error = NULL;

  if (!unformat_user (input, unformat_line_input, line_input))
    return 0;

  while (unformat_check_input (line_input) != UNFORMAT_END_OF_INPUT)
    {
      if (unformat (line_input, "interface pg%u", &if_id))
    ;
      else if (unformat (line_input, "gso-enabled"))
    {
      gso_enabled = 1;
      if (unformat (line_input, "gso-size %u", &gso_size))
        ;
      else
        {
          error = clib_error_create ("gso enabled but gso size missing");
          goto done;
        }
    }
      else
    {
      error = clib_error_create ("unknown input `%U'",
                     format_unformat_error, line_input);
      goto done;
    }
    }

  pg_interface_add_or_get (pg, if_id, gso_enabled, gso_size);

done:
  unformat_free (line_input);

  return error;
}
```

**delete packet generator**
```
VLIB_CLI_COMMAND (del_stream_cli, static) = {
  .path = "packet-generator delete",
  .function = del_stream,
  .short_help = "Delete stream with given name",
};

static clib_error_t *
del_stream (vlib_main_t * vm,
        unformat_input_t * input, vlib_cli_command_t * cmd)
{
  pg_main_t *pg = &pg_main;
  u32 i;

  if (!unformat (input, "%U",
         &unformat_hash_vec_string, pg->stream_index_by_name, &i))
    return clib_error_create ("expected stream name `%U'",
                  format_unformat_error, input);

  pg_stream_del (pg, i);
  return 0;
}
```

**packet generator capture**
```
VLIB_CLI_COMMAND (pg_capture_cmd, static) = {
  .path = "packet-generator capture",
  .short_help = "packet-generator capture <interface name> pcap <filename> [count <n>]",
  .function = pg_capture_cmd_fn,
};

static clib_error_t *
pg_capture_cmd_fn (vlib_main_t * vm,
           unformat_input_t * input, vlib_cli_command_t * cmd)
{
  clib_error_t *error = 0;
  vnet_main_t *vnm = vnet_get_main ();
  unformat_input_t _line_input, *line_input = &_line_input;
  vnet_hw_interface_t *hi = 0;
  u8 *pcap_file_name = 0;
  u32 hw_if_index;
  u32 is_disable = 0;
  u32 count = ~0;

  if (!unformat_user (input, unformat_line_input, line_input))
    return 0;

  while (unformat_check_input (line_input) != UNFORMAT_END_OF_INPUT)
    {
      if (unformat (line_input, "%U",
            unformat_vnet_hw_interface, vnm, &hw_if_index))
    {
      hi = vnet_get_hw_interface (vnm, hw_if_index);
    }

      else if (unformat (line_input, "pcap %s", &pcap_file_name))
    ;
      else if (unformat (line_input, "count %u", &count))
    ;
      else if (unformat (line_input, "disable"))
    is_disable = 1;

      else
    {
      error = clib_error_create ("unknown input `%U'",
                     format_unformat_error, line_input);
      goto done;
    }
    }

  if (!hi)
    {
      error = clib_error_return (0, "Please specify interface name");
      goto done;
    }

  if (hi->dev_class_index != pg_dev_class.index)
    {
      error =
    clib_error_return (0, "Please specify packet-generator interface");
      goto done;
    }

  if (!pcap_file_name && is_disable == 0)
    {
      error = clib_error_return (0, "Please specify pcap file name");
      goto done;
    }


  pg_capture_args_t _a, *a = &_a;

  a->hw_if_index = hw_if_index;
  a->dev_instance = hi->dev_instance;
  a->is_enabled = !is_disable;
  a->pcap_file_name = pcap_file_name;
  a->count = count;

  error = pg_capture (a);

done:
  unformat_free (line_input);

  return error;
}
```

**packet generator configure**
```
VLIB_CLI_COMMAND (change_stream_parameters_cli, static) = {
  .path = "packet-generator configure",
  .short_help = "Change packet generator stream parameters",
  .function = change_stream_parameters,
};

static clib_error_t *
change_stream_parameters (vlib_main_t * vm,
              unformat_input_t * input, vlib_cli_command_t * cmd)
{
  pg_main_t *pg = &pg_main;
  pg_stream_t *s, s_new;
  u32 stream_index = ~0;
  clib_error_t *error;

  if (unformat (input, "%U", unformat_hash_vec_string,
        pg->stream_index_by_name, &stream_index))
    ;
  else
    return clib_error_create ("expecting stream name; got `%U'",
                  format_unformat_error, input);

  s = pool_elt_at_index (pg->streams, stream_index);
  s_new = s[0];

  while (unformat_check_input (input) != UNFORMAT_END_OF_INPUT)
    {
      if (unformat_user (input, unformat_pg_stream_parameter, &s_new))
    ;

      else
    return clib_error_create ("unknown input `%U'",
                  format_unformat_error, input);
    }

  error = validate_stream (&s_new);
  if (!error)
    {
      s[0] = s_new;
      pg_stream_change (pg, s);
    }

  return error;
}
```

**show packet generator**
```
VLIB_CLI_COMMAND (show_streams_cli, static) = {
  .path = "show packet-generator ",
  .short_help = "show packet-generator [verbose]",
  .function = show_streams,
};

static clib_error_t *
show_streams (vlib_main_t * vm,
          unformat_input_t * input, vlib_cli_command_t * cmd)
{
  pg_main_t *pg = &pg_main;
  pg_stream_t *s;
  int verbose = 0;

  while (unformat_check_input (input) != UNFORMAT_END_OF_INPUT)
    {
      if (unformat (input, "verbose"))
    verbose = 1;
      else
    break;
    }

  if (pool_elts (pg->streams) == 0)
    {
      vlib_cli_output (vm, "no streams currently defined");
      goto done;
    }

  vlib_cli_output (vm, "%U", format_pg_stream, 0, 0);
  /* *INDENT-OFF* */
  pool_foreach (s, pg->streams, ({
      vlib_cli_output (vm, "%U", format_pg_stream, s, verbose);
    }));
  /* *INDENT-ON* */

done:
  return 0;
}
```
