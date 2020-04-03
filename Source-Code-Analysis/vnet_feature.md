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

### 注册

#### vnet/feature/registration.c

```
/**
 * @file
 * @brief Feature Subgraph Ordering.

    Dynamically compute feature subgraph ordering by performing a
    topological sort across a set of "feature A before feature B" and
    "feature C after feature B" constraints.

    Use the topological sort result to set up vnet_config_main_t's for
    use at runtime.

    Feature subgraph arcs are simple enough. They start at specific
    fixed nodes, and end at specific fixed nodes.  In between, a
    per-interface current feature configuration dictates which
    additional nodes each packet visits. Each so-called feature node
    can [of course] drop any specific packet.

    See ip4_forward.c, ip6_forward.c in this directory to see the
    current rx-unicast, rx-multicast, and tx feature subgraph arc
    definitions.

    Let's say that we wish to add a new feature to the ip4 unicast
    feature subgraph arc, which needs to run before @c ip4-lookup.  In
    either base code or a plugin,
    <CODE><PRE>
    \#include <vnet/feature/feature.h>
    </PRE></CODE>

    and add the new feature as shown:

    <CODE><PRE>
    VNET_FEATURE_INIT (ip4_lookup, static) =
    {
      .arch_name = "ip4-unicast",
      .node_name = "my-ip4-unicast-feature",
      .runs_before = VLIB_FEATURES ("ip4-lookup")
    };
    </PRE></CODE>

    Here's the standard coding pattern to enable / disable
    @c my-ip4-unicast-feature on an interface:

    <CODE><PRE>

    sw_if_index = <interface-handle>
    vnet_feature_enable_disable ("ip4-unicast", "my-ip4-unicast-feature",
                                 sw_if_index, 1 );
    </PRE></CODE>

    Here's how to obtain the correct next node index in packet
    processing code, aka in the implementation of @c my-ip4-unicast-feature:

    <CODE><PRE>
    vnet_feature_next (sw_if_index0, &next0, b0);

    </PRE></CODE>

    Nodes are free to drop or otherwise redirect packets. Packets
    which "pass" should be enqueued via the next0 arc computed by
    vnet_feature_next.
*/


static int
comma_split (u8 * s, u8 ** a, u8 ** b)
{
  *a = s;

  while (*s && *s != ',')
    s++;

  if (*s == ',')
    *s = 0;
  else
    return 1;

  *b = (u8 *) (s + 1);
  return 0;
}

/**
 * @brief Initialize a feature graph arc
 * @param vm vlib main structure pointer
 * @param vcm vnet config main structure pointer
 * @param feature_start_nodes names of start-nodes which use this
 *      feature graph arc
 * @param num_feature_start_nodes number of start-nodes
 * @param first_reg first element in
 *        [an __attribute__((constructor)) function built, or
 *        otherwise created] singly-linked list of feature registrations
 * @param first_const first element in
 *        [an __attribute__((constructor)) function built, or
 *        otherwise created] singly-linked list of bulk order constraints
 * @param [out] in_feature_nodes returned vector of
 *        topologically-sorted feature node names, for use in
 *        show commands
 * @returns 0 on success, otherwise an error message. Errors
 *        are fatal since they invariably involve mistyped node-names, or
 *        genuinely missing node-names
 */
clib_error_t *
vnet_feature_arc_init (vlib_main_t * vm,
               vnet_config_main_t * vcm,
               char **feature_start_nodes,
               int num_feature_start_nodes,
               char *last_in_arc,
               vnet_feature_registration_t * first_reg,
               vnet_feature_constraint_registration_t *
               first_const_set, char ***in_feature_nodes)
{
  uword *index_by_name;
  uword *reg_by_index;
  u8 **node_names = 0;
  u8 *node_name;
  char *prev_name;
  char **these_constraints;
  char *this_constraint_c;
  u8 **constraints = 0;
  u8 *constraint_tuple;
  u8 *this_constraint;
  u8 **orig, **closure;
  uword *p;
  int i, j, k;
  u8 *a_name, *b_name;
  int a_index, b_index;
  int n_features;
  u32 *result = 0;
  vnet_feature_registration_t *this_reg = 0;
  vnet_feature_constraint_registration_t *this_const_set = 0;
  char **feature_nodes = 0;
  hash_pair_t *hp;
  u8 **keys_to_delete = 0;

  index_by_name = hash_create_string (0, sizeof (uword));
  reg_by_index = hash_create (0, sizeof (uword));

  this_reg = first_reg;

  /* Autogenerate <node> before <last-in-arc> constraints */
  if (last_in_arc)
    {
      while (this_reg)
    {
      /* If this isn't the last node in the arc... */
      if (clib_strcmp (this_reg->node_name, last_in_arc))
        {
          /*
           * Add an explicit constraint so this feature will run
           * before the last node in the arc
           */
          constraint_tuple = format (0, "%s,%s%c", this_reg->node_name,
                     last_in_arc, 0);
          vec_add1 (constraints, constraint_tuple);
        }
      this_reg = this_reg->next_in_arc;
    }
      this_reg = first_reg;
    }

  /* pass 1, collect feature node names, construct a before b pairs */
  while (this_reg)
    {
      node_name = format (0, "%s%c", this_reg->node_name, 0);
      hash_set (reg_by_index, vec_len (node_names), (uword) this_reg);

      hash_set_mem (index_by_name, node_name, vec_len (node_names));

      vec_add1 (node_names, node_name);

      these_constraints = this_reg->runs_before;
      while (these_constraints && these_constraints[0])
    {
      this_constraint_c = these_constraints[0];

      constraint_tuple = format (0, "%s,%s%c", node_name,
                     this_constraint_c, 0);
      vec_add1 (constraints, constraint_tuple);
      these_constraints++;
    }

      these_constraints = this_reg->runs_after;
      while (these_constraints && these_constraints[0])
    {
      this_constraint_c = these_constraints[0];

      constraint_tuple = format (0, "%s,%s%c",
                     this_constraint_c, node_name, 0);
      vec_add1 (constraints, constraint_tuple);
      these_constraints++;
    }

      this_reg = this_reg->next_in_arc;
    }

  /* pass 2, collect bulk "a then b then c then d" constraints */
  this_const_set = first_const_set;
  while (this_const_set)
    {
      these_constraints = this_const_set->node_names;

      prev_name = 0;
      /* Across the list of constraints */
      while (these_constraints && these_constraints[0])
    {
      this_constraint_c = these_constraints[0];
      p = hash_get_mem (index_by_name, this_constraint_c);
      if (p == 0)
        {
          clib_warning
        ("bulk constraint feature node '%s' not found for arc '%s'",
         this_constraint_c);
          these_constraints++;
          continue;
        }

      if (prev_name == 0)
        {
          prev_name = this_constraint_c;
          these_constraints++;
          continue;
        }

      constraint_tuple = format (0, "%s,%s%c", prev_name,
                     this_constraint_c, 0);
      vec_add1 (constraints, constraint_tuple);
      prev_name = this_constraint_c;
      these_constraints++;
    }

      this_const_set = this_const_set->next_in_arc;
    }

  n_features = vec_len (node_names);
  orig = clib_ptclosure_alloc (n_features);

  for (i = 0; i < vec_len (constraints); i++)
    {
      this_constraint = constraints[i];

      if (comma_split (this_constraint, &a_name, &b_name))
    return clib_error_return (0, "comma_split failed!");

      p = hash_get_mem (index_by_name, a_name);
      /*
       * Note: the next two errors mean that something is
       * b0rked. As in: if you code "A depends on B," and you forget
       * to define a FEATURE_INIT macro for B, you lose.
       * Nonexistent graph nodes are tolerated.
       */
      if (p == 0)
    {
      clib_warning ("feature node '%s' not found (before '%s', arc '%s')",
            a_name, b_name, first_reg->arc_name);
      continue;
    }
      a_index = p[0];

      p = hash_get_mem (index_by_name, b_name);
      if (p == 0)
    {
      clib_warning ("feature node '%s' not found (after '%s', arc '%s')",
            b_name, a_name, first_reg->arc_name);
      continue;
    }
      b_index = p[0];

      /* add a before b to the original set of constraints */
      orig[a_index][b_index] = 1;
      vec_free (this_constraint);
    }

  /* Compute the positive transitive closure of the original constraints */
  closure = clib_ptclosure (orig);

  /* Compute a partial order across feature nodes, if one exists. */
again:
  for (i = 0; i < n_features; i++)
    {
      for (j = 0; j < n_features; j++)
    {
      if (closure[i][j])
        goto item_constrained;
    }
      /* Item i can be output */
      vec_add1 (result, i);
      {
    for (k = 0; k < n_features; k++)
      closure[k][i] = 0;
    /*
     * Add a "Magic" a before a constraint.
     * This means we'll never output it again
     */
    closure[i][i] = 1;
    goto again;
      }
    item_constrained:
      ;
    }

  /* see if we got a partial order... */
  if (vec_len (result) != n_features)
    return clib_error_return
      (0, "Arc '%s': failed to find a suitable feature order!",
       first_reg->arc_name);

  /*
   * We win.
   * Bind the index variables, and output the feature node name vector
   * using the partial order we just computed. Result is in stack
   * order, because the entry with the fewest constraints (e.g. none)
   * is output first, etc.
   */

  for (i = n_features - 1; i >= 0; i--)
    {
      p = hash_get (reg_by_index, result[i]);
      ASSERT (p != 0);
      this_reg = (vnet_feature_registration_t *) p[0];
      if (this_reg->feature_index_ptr)
    *this_reg->feature_index_ptr = n_features - (i + 1);
      this_reg->feature_index = n_features - (i + 1);
      vec_add1 (feature_nodes, this_reg->node_name);
    }

  /* Set up the config infrastructure */
  vnet_config_init (vm, vcm,
            feature_start_nodes,
            num_feature_start_nodes,
            feature_nodes, vec_len (feature_nodes));

  /* Save a copy for show command */
  *in_feature_nodes = feature_nodes;

  /* Finally, clean up all the shit we allocated */
  /* *INDENT-OFF* */
  hash_foreach_pair (hp, index_by_name,
  ({
    vec_add1 (keys_to_delete, (u8 *)hp->key);
  }));
  /* *INDENT-ON* */
  hash_free (index_by_name);
  for (i = 0; i < vec_len (keys_to_delete); i++)
    vec_free (keys_to_delete[i]);
  vec_free (keys_to_delete);
  hash_free (reg_by_index);
  vec_free (result);
  clib_ptclosure_free (orig);
  clib_ptclosure_free (closure);
  return 0;
}
```
