## 配置
**ipfix-export/flow_report.c**

```
/* *INDENT-OFF* */
VLIB_CLI_COMMAND (set_ipfix_exporter_command, static) = {
    .path = "set ipfix exporter",
    .short_help = "set ipfix exporter "
                  "collector <ip4-address> [port <port>] "
                  "src <ip4-address> [fib-id <fib-id>] "
                  "[path-mtu <path-mtu>] "
                  "[template-interval <template-interval>]",
                  "[udp-checksum]",
    .function = set_ipfix_exporter_command_fn,
};
/* *INDENT-ON* */
```

```
/* *INDENT-OFF* */
VLIB_CLI_COMMAND (ipfix_flush_command, static) = {
    .path = "ipfix flush",
    .short_help = "flush the current ipfix data [for make test]",
    .function = ipfix_flush_command_fn,
};
/* *INDENT-ON* */
```

**ipfix-export/flow_report_clissify.c**
```
/* *INDENT-OFF* */
VLIB_CLI_COMMAND (ipfix_classify_table_add_del_command, static) = {
  .path = "ipfix classify table",
  .short_help = "ipfix classify table add|del <table-index>",
  .function = ipfix_classify_table_add_del_command_fn,
};
/* *INDENT-ON* */

/* *INDENT-OFF* */
VLIB_CLI_COMMAND (set_ipfix_classify_stream_command, static) = {
  .path = "set ipfix classify stream",
  .short_help = "set ipfix classify stream"
                "[domain <domain-id>] [src-port <src-port>]",
  .function = set_ipfix_classify_stream_command_fn,
};
/* *INDENT-ON* */
```

## 处理
**ipfix-export/flow_report.c**
```
/* *INDENT-OFF* */
VLIB_REGISTER_NODE (flow_report_process_node) = {
    .function = flow_report_process,
    .type = VLIB_NODE_TYPE_PROCESS,
    .name = "flow-report-process",
};
/* *INDENT-ON* */
```
```

```
