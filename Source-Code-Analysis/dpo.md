## dpo

dpo(Data-Path Object)是一个代表动作的对象，它应用于用户转发数据包的vpp数据路径(data-path)上。

### vnet/dpo/dpo.h

```
typedef enum dpo_proto_t_
{
    DPO_PROTO_IP4 = 0,
    DPO_PROTO_IP6,
	/* 多协议标签交换(MPLS) */
    DPO_PROTO_MPLS,
    DPO_PROTO_ETHERNET,
	/* 新型组播技术Bier, see rfc: https://tools.ietf.org/html/rfc8279 */
    DPO_PROTO_BIER,
	/* NSH(Network Service Header)协议，see rfc: https://datatracker.ietf.org/doc/rfc8300/?include_text=1 */
    DPO_PROTO_NSH,
} __attribute__((packed)) dpo_proto_t;
```

```
/**
 * @brief Common types of data-path objects
 * New types can be dynamically added using dpo_register_new_type()
 */
typedef enum dpo_type_t_ {
    /**
     * A non-zero value first so we can spot unitialisation errors
     */
    DPO_FIRST,
    DPO_DROP,
    DPO_IP_NULL,
    DPO_PUNT,
    /**
     * @brief load-balancing over a choice of [un]equal cost paths
     */
    DPO_LOAD_BALANCE,
    DPO_REPLICATE,
    DPO_ADJACENCY,
    DPO_ADJACENCY_INCOMPLETE,
    DPO_ADJACENCY_MIDCHAIN,
    DPO_ADJACENCY_GLEAN,
    DPO_ADJACENCY_MCAST,
    DPO_ADJACENCY_MCAST_MIDCHAIN,
    DPO_RECEIVE,
    DPO_LOOKUP,
    DPO_LISP_CP,
    DPO_CLASSIFY,
    DPO_MPLS_DISPOSITION_PIPE,
    DPO_MPLS_DISPOSITION_UNIFORM,
    DPO_MFIB_ENTRY,
    DPO_INTERFACE_RX,
    DPO_INTERFACE_TX,
    DPO_DVR,
    DPO_L3_PROXY,
    DPO_BIER_TABLE,
    DPO_BIER_FMASK,
    DPO_BIER_IMP,
    DPO_BIER_DISP_TABLE,
    DPO_BIER_DISP_ENTRY,
    DPO_IP6_LL,
    DPO_PW_CW,
    DPO_LAST,
} __attribute__((packed)) dpo_type_t;
```

```
/**
 * @brief The identity of a DPO is a combination of its type and its
 * instance number/index of objects of that type
 */
typedef struct dpo_id_t_ {
    /**
     * dpo类型
     */
    dpo_type_t dpoi_type;
    /**
     * 数据路径(data-path)协议
     */
    dpo_proto_t dpoi_proto;
    /**
     * 下一个node
     */
    u16 dpoi_next_node;
    /**
     * dpo标识(u32)
     */
    index_t dpoi_index;
} __attribute__ ((aligned(sizeof(u64)))) dpo_id_t;

```

```
/**
 * @brief An initialiser for DPOs declared on the stack.
 * Thenext node is set to 0 since VLIB graph nodes should set 0 index to drop.
 */
#define DPO_INVALID                \
{                                  \
    .dpoi_type = DPO_FIRST,        \
    .dpoi_proto = DPO_PROTO_NONE,  \
    .dpoi_index = INDEX_INVALID,   \
    .dpoi_next_node = 0,           \
}
```

```
/**
 * @brief Return true if the DPO object is valid, i.e. has been initialised.
 */
static inline int
dpo_id_is_valid (const dpo_id_t *dpoi)
{
    /* 检查：dpo类型和标识 */
    return (dpoi->dpoi_type != DPO_FIRST &&
	    dpoi->dpoi_index != INDEX_INVALID);
}
```

```
void
dpo_lock (dpo_id_t *dpo)
{
    if (!dpo_id_is_valid(dpo))
	return;

    dpo_vfts[dpo->dpoi_type].dv_lock(dpo);
}

void
dpo_unlock (dpo_id_t *dpo)
{
    if (!dpo_id_is_valid(dpo))
	return;

    dpo_vfts[dpo->dpoi_type].dv_unlock(dpo);
}
```