## DPDK

### dpdk device

DPDK更新计数器，包括rx_nombuf,rx_imissed,rx_errors。

```
uint64_t imissed
Total of RX packets dropped by the HW, because there are no available buffer (i.e. RX queues are full).

Definition at line 248 of file rte_ethdev.h.

uint64_t rx_nombuf
Total number of RX mbuf allocation failures.

Definition at line 254 of file rte_ethdev.h.

uint64_t ierrors
Total number of erroneous received packets.

Definition at line 252 of file rte_ethdev.h.
```

```
RX-missed

Total of RX packets dropped by the HW, because there are no available buffer (i.e. RX queues are full).

The main reason for full RX queues is a "slow" application, which is not able to process packets in a rate they arrive on interface.

RX-errors

Total number of erroneous received packets, i.e. packets with incorrect checksum, runts, giants etc.

RX-nombuf

Total number of RX mbuf allocation failures, i.e. RX packet was drop due to lack of free mbufs in the mempool.
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

  /* update上次更新时间 */
  xd->time_last_stats_update = now ? now : xd->time_last_stats_update;
  /* update上次统计信息 */
  clib_memcpy_fast (&xd->last_stats, &xd->stats, sizeof (xd->last_stats));
  /* 根据dpdk设备端口号获取设备统计信息 */
  rte_eth_stats_get (xd->port_id, &xd->stats);

  /* maybe bump interface rx no buffer counter */
  /* 本次统计信息中mbuf申请失败的次数跟上次不一样 */
  if (PREDICT_FALSE (xd->stats.rx_nombuf != xd->last_stats.rx_nombuf))
    {
      /* 获取接口的nombuf简单计数器 */
      cm = vec_elt_at_index (vnm->interface_main.sw_if_counters,
			     VNET_INTERFACE_COUNTER_RX_NO_BUF);
     
      /* 本次nombuf减去上次nombuf，然后累加，即当前nombuf的统计信息 */
      vlib_increment_simple_counter (cm, thread_index, xd->sw_if_index,
				     xd->stats.rx_nombuf -
				     xd->last_stats.rx_nombuf);
    }

  /* missed pkt counter */
  /* 本次统计信息中丢包数跟上次不一样 */
  if (PREDICT_FALSE (xd->stats.imissed != xd->last_stats.imissed))
    {
      /* 获取接口的imissed简单计数器 */
      cm = vec_elt_at_index (vnm->interface_main.sw_if_counters,
			     VNET_INTERFACE_COUNTER_RX_MISS);
      /* 本次imissed减去上次imissed，然后累加，即当前imissed的统计信息 */
      vlib_increment_simple_counter (cm, thread_index, xd->sw_if_index,
				     xd->stats.imissed -
				     xd->last_stats.imissed);
    }
  
  /* 本次统计信息中包错误数跟上次不一样 */
  rxerrors = xd->stats.ierrors;
  last_rxerrors = xd->last_stats.ierrors;
  if (PREDICT_FALSE (rxerrors != last_rxerrors))
    {
      /* 获取接口的ierrors简单计数器 */
      cm = vec_elt_at_index (vnm->interface_main.sw_if_counters,
			     VNET_INTERFACE_COUNTER_RX_ERROR);

      /* 本次ierrors减去上次ierrors，然后累加，即当前ierrors的统计信息 */
      vlib_increment_simple_counter (cm, thread_index, xd->sw_if_index,
				     rxerrors - last_rxerrors);
    }

  /* 获取rte_eth_stats之外的统计信息 */
  dpdk_get_xstats (xd);
}
```

