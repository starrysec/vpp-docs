## device

### dpdk device

DPDK更新计数器，包括rx_nombuf,rx_imissed,rx_errors。

```
uint64_t imissed
Total of RX packets dropped by the HW, because there are no available buffer (i.e. RX queues are full).

Definition at line 248 of file rte_ethdev.h.
```

```
uint64_t rx_nombuf
Total number of RX mbuf allocation failures.

Definition at line 254 of file rte_ethdev.h.
```

```
uint64_t ierrors
Total number of erroneous received packets.

Definition at line 252 of file rte_ethdev.h.
```

```
static inline void
dpdk_update_counters (dpdk_device_t * xd, f64 now)
{
  vlib_simple_counter_main_t *cm;
  vnet_main_t *vnm = vnet_get_main ();
  u32 thread_index = vlib_get_thread_index ();
  u64 rxerrors, last_rxerrors;

  /* only update counters for PMD interfaces */
  if ((xd->flags & DPDK_DEVICE_FLAG_PMD) == 0)
    return;

  xd->time_last_stats_update = now ? now : xd->time_last_stats_update;
  clib_memcpy_fast (&xd->last_stats, &xd->stats, sizeof (xd->last_stats));
  rte_eth_stats_get (xd->port_id, &xd->stats);

  /* maybe bump interface rx no buffer counter */
  if (PREDICT_FALSE (xd->stats.rx_nombuf != xd->last_stats.rx_nombuf))
    {
      cm = vec_elt_at_index (vnm->interface_main.sw_if_counters,
			     VNET_INTERFACE_COUNTER_RX_NO_BUF);

      vlib_increment_simple_counter (cm, thread_index, xd->sw_if_index,
				     xd->stats.rx_nombuf -
				     xd->last_stats.rx_nombuf);
    }

  /* missed pkt counter */
  if (PREDICT_FALSE (xd->stats.imissed != xd->last_stats.imissed))
    {
      cm = vec_elt_at_index (vnm->interface_main.sw_if_counters,
			     VNET_INTERFACE_COUNTER_RX_MISS);

      vlib_increment_simple_counter (cm, thread_index, xd->sw_if_index,
				     xd->stats.imissed -
				     xd->last_stats.imissed);
    }
  rxerrors = xd->stats.ierrors;
  last_rxerrors = xd->last_stats.ierrors;

  if (PREDICT_FALSE (rxerrors != last_rxerrors))
    {
      cm = vec_elt_at_index (vnm->interface_main.sw_if_counters,
			     VNET_INTERFACE_COUNTER_RX_ERROR);

      vlib_increment_simple_counter (cm, thread_index, xd->sw_if_index,
				     rxerrors - last_rxerrors);
    }

  dpdk_get_xstats (xd);
}
```

