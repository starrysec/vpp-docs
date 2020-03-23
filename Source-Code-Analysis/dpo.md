## dpo

dpo(Data-Path Object)是一个代表动作的对象(基类)，它应用于用户转发数据包的vpp数据路径(data-path)上。
see: https://www.asumu.xyz/blog/2018/11/14/data-path-objects-in-vpp/

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
     * dpo索引(u32)
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
    /* 检查：dpo类型和索引 */
    return (dpoi->dpoi_type != DPO_FIRST &&
	    dpoi->dpoi_index != INDEX_INVALID);
}
```

```
void
dpo_lock (dpo_id_t *dpo)
{
    /* 检查dpo是否有效 */
    if (!dpo_id_is_valid(dpo))
	return;

	/* 增加dpo锁引用计数（子类具体实现） */
    dpo_vfts[dpo->dpoi_type].dv_lock(dpo);
}

void
dpo_unlock (dpo_id_t *dpo)
{
	/* 检查dpo是否有效 */
    if (!dpo_id_is_valid(dpo))
	return;

	/* 减少dpo锁引用计数（子类具体实现） */
    dpo_vfts[dpo->dpoi_type].dv_unlock(dpo);
}

void
dpo_mk_interpose (const dpo_id_t *original,
                  const dpo_id_t *parent,
                  dpo_id_t *clone)
{
	/* 检查dpo是否有效 */
    if (!dpo_id_is_valid(original))
	return;

	/* 向子类发送父类改变的信号 */
    dpo_vfts[original->dpoi_type].dv_mk_interpose(original, parent, clone);
}

void
dpo_set (dpo_id_t *dpo,
	 dpo_type_t type,
	 dpo_proto_t proto,
	 index_t index)
{
    dpo_id_t tmp = *dpo;

    dpo->dpoi_type = type;
    dpo->dpoi_proto = proto,
    dpo->dpoi_index = index;

	/* 检查dpo类型是否adjacency类型 */
    if (DPO_ADJACENCY == type)
    {
		/*
		 * set the adj subtype
		 */
		ip_adjacency_t *adj;
		
		/* 根据dpo索引，获取adj对象 */
		adj = adj_get(index);
		
		/* ip4-lookup后的下一个节点 */
		switch (adj->lookup_next_index)
		{
		/* 匹配到不完整的邻接表(incomplete adjacency)，需要使用arp协议发现目的地址的rewrite字符串 */
		case IP_LOOKUP_NEXT_ARP:
			dpo->dpoi_type = DPO_ADJACENCY_INCOMPLETE;
			break;
		/* 匹配到Multicast Midchain Adjacency. An Adjacency for sending macst packets on a tunnel/virtual interface */
		case IP_LOOKUP_NEXT_MIDCHAIN:
			dpo->dpoi_type = DPO_ADJACENCY_MIDCHAIN;
			break;
		/* 匹配到mid-chain adjacency */
		case IP_LOOKUP_NEXT_MCAST_MIDCHAIN:
			dpo->dpoi_type = DPO_ADJACENCY_MCAST_MIDCHAIN;
			break;
		/* 匹配到Multicast Adjacency */
		case IP_LOOKUP_NEXT_MCAST:
			dpo->dpoi_type = DPO_ADJACENCY_MCAST;
				break;
		/* 匹配到不完整的邻接表(interface route)，需要使用arp协议发现目的地址的rewrite字符串*/
		case IP_LOOKUP_NEXT_GLEAN:
			dpo->dpoi_type = DPO_ADJACENCY_GLEAN;
			break;
		default:
			break;
		}
    }
	/* 增加dpo锁引用计数 */
    dpo_lock(dpo);
	/* 减少tmp锁引用计数 */
    dpo_unlock(&tmp);
}

void
dpo_reset (dpo_id_t *dpo)
{
    dpo_id_t tmp = DPO_INVALID;

    /*
     * use the atomic copy operation.
     */
    dpo_copy(dpo, &tmp);
}

/**
 * \brief
 * Compare two Data-path objects
 *
 * like memcmp, return 0 is matching, !0 otherwise.
 */
int
dpo_cmp (const dpo_id_t *dpo1,
	 const dpo_id_t *dpo2)
{
    int res;

    res = dpo1->dpoi_type - dpo2->dpoi_type;

    if (0 != res) return (res);

    return (dpo1->dpoi_index - dpo2->dpoi_index);
}

void
dpo_copy (dpo_id_t *dst,
	  const dpo_id_t *src)
{
    dpo_id_t tmp = *dst;

    /*
     * the destination is written in a single u64 write - hence atomically w.r.t
     * any packets inflight.
     */
    *((u64*)dst) = *(u64*)src;

    dpo_lock(dst);
    dpo_unlock(&tmp);
}
```