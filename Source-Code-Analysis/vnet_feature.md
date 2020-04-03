## feature

### 配置

#### vnet/feature/feature.c

**set feature for given interface**

```
/*?
 * Set feature for given interface
 *
 * @cliexpar
 * Example:
 * @cliexcmd{set interface feature GigabitEthernet2/0/0 ip4_flow_classify arc ip4_unicast}
 * @cliexend
 * @endparblock
?*/
VLIB_CLI_COMMAND (set_interface_feature_command, static) = {
  .path = "set interface feature",
  .short_help = "set interface feature <intfc> <feature_name> arc <arc_name> "
      "[disable]",
  .function = set_interface_features_command_fn,
};

static clib_error_t *
set_interface_features_command_fn (vlib_main_t * vm,
                   unformat_input_t * input,
                   vlib_cli_command_t * cmd)
{
  vnet_main_t *vnm = vnet_get_main ();
  unformat_input_t _line_input, *line_input = &_line_input;
  clib_error_t *error = 0;
  
  u8 *arc_name = 0;
  u8 *feature_name = 0;
  u32 sw_if_index = ~0;
  u8 enable = 1;

  /* Get a line of input. */
  if (!unformat_user (input, unformat_line_input, line_input))
    return 0;

  while (unformat_check_input (line_input) != UNFORMAT_END_OF_INPUT)
  {
    // parse interface, feature name and arc name.
    if (unformat(line_input, "%U %s arc %s", unformat_vnet_sw_interface, vnm,
       &sw_if_index, &feature_name, &arc_name));
    // enable or disable feature.
    else if (unformat (line_input, "disable"))
      enable = 0;
    // error occurred.
    else
    {
      error = unformat_parse_error (line_input);
      goto done;
    }
  }
  if (!feature_name || !arc_name)
  {
      error = clib_error_return (0, "Both feature name and arc required...");
      goto done;
  }

  if (sw_if_index == ~0)
  {
      error = clib_error_return (0, "Interface not specified...");
      goto done;
  }

  // init arc name and feature name vector, add 0 to the end of the vector.
  vec_add1 (arc_name, 0);
  vec_add1 (feature_name, 0);

  // get arc index by arc name.
  u8 arc_index;
  arc_index = vnet_get_feature_arc_index ((const char *) arc_name);
  if (arc_index == (u8) ~ 0)
  {
      error =
    clib_error_return (0, "Unknown arc name (%s)... ",
               (const char *) arc_name);
      goto done;
  }

  // get feature registration info by arc name and feature name.
  vnet_feature_registration_t *reg;
  reg = vnet_get_feature_reg ((const char *) arc_name, (const char *) feature_name);
  if (reg == 0)
  {
      error =
    clib_error_return (0,
               "Feature (%s) not registered to arc (%s)... See 'show features verbose' for valid feature/arc combinations. ",
               feature_name, arc_name);
      goto done;
  }
  // callback function to enable/disable feature.
  if (reg->enable_disable_cb)
    error = reg->enable_disable_cb (sw_if_index, enable);
  // enable/disable feature for given interface.
  if (!error)
    vnet_feature_enable_disable ((const char *) arc_name,
                 (const char *) feature_name, sw_if_index,
                 enable, 0, 0);

done:
  vec_free (feature_name);
  vec_free (arc_name);
  unformat_free (line_input);
  return error;
}
```

### 查看

#### vnet/feature/feature.c

**show features**
```
/*?
 * Display the set of available driver features
 *
 * @cliexpar
 * Example:
 * @cliexcmd{show features [verbose]}
 * @cliexend
 * @endparblock
?*/
VLIB_CLI_COMMAND (show_features_command, static) = {
  .path = "show features",
  .short_help = "show features [verbose]",
  .function = show_features_command_fn,
};

/** Display the set of available driver features.
    Useful for verifying that expected features are present
*/
static clib_error_t *
show_features_command_fn (vlib_main_t * vm,
              unformat_input_t * input, vlib_cli_command_t * cmd)
{
  vnet_feature_main_t *fm = &feature_main;
  vnet_feature_arc_registration_t *areg;
  vnet_feature_registration_t *freg;
  vnet_feature_registration_t *feature_regs = 0;
  int verbose = 0;

  if (unformat (input, "verbose"))
    verbose = 1;

  vlib_cli_output (vm, "Available feature paths");

  // 遍历arc
  areg = fm->next_arc;
  while (areg)
  {
    // show both arc index and arc name when verbose flag set.
    if (verbose)
      vlib_cli_output (vm, "[%2d] %s:", areg->feature_arc_index,
             areg->arc_name);
    // shwo arc name.
    else
      vlib_cli_output (vm, "%s:", areg->arc_name);

    // featre registration info of arc's next feature.
    freg = fm->next_feature_by_arc[areg->feature_arc_index];
    while (freg)
    {
      // add feature to feature_regs vector.
      vec_add1 (feature_regs, freg[0]);
      freg = freg->next_in_arc;
    }

    // 给feature_regs排序
    vec_sort_with_function (feature_regs, feature_cmp);

    // 遍历featrue_args
    vec_foreach (freg, feature_regs)
    {
      // 输出feature index和feature节点名称
      if (verbose)
        vlib_cli_output (vm, "  [%2d]: %s\n", freg->feature_index,
               freg->node_name);
      else
        vlib_cli_output (vm, "  %s\n", freg->node_name);
    }
    vec_reset_length (feature_regs);
    /* next arc */
    areg = areg->next;
  }
  vec_free (feature_regs);

  return 0;
}
```
