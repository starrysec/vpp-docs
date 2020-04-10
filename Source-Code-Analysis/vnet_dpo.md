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

### dpo初始化

dpo初始化(dpo_module_init)时会调用具体的dpo初始化函数(xxx_dpo_module_init)，具体的dpo初始化函数中调用dpo_register或dpo_register_new_type注册自己到dpo。

```
VLIB_INIT_FUNCTION(dpo_module_init) =
{
    .runs_before = VLIB_INITS ("ip_main_init"),
};

static clib_error_t *
dpo_module_init (vlib_main_t * vm)
{
    drop_dpo_module_init();
    punt_dpo_module_init();
    receive_dpo_module_init();
    load_balance_module_init();
    mpls_label_dpo_module_init();
    classify_dpo_module_init();
    lookup_dpo_module_init();
    ip_null_dpo_module_init();
    ip6_ll_dpo_module_init();
    replicate_module_init();
    interface_rx_dpo_module_init();
    interface_tx_dpo_module_init();
    mpls_disp_dpo_module_init();
    dvr_dpo_module_init();
    l3_proxy_dpo_module_init();
    pw_cw_dpo_module_init();

    return (NULL);
}
```

### dpo注册

**例子**

```
const static dpo_vft_t adj_nbr_dpo_vft = {
    .dv_lock = adj_dpo_lock,
    .dv_unlock = adj_dpo_unlock,
    .dv_format = format_adj_nbr,
    .dv_mem_show = adj_mem_show,
    .dv_get_urpf = adj_dpo_get_urpf,
};

const static char* const * const nbr_nodes[DPO_PROTO_NUM] =
{
    [DPO_PROTO_IP4]  = nbr_ip4_nodes,
    [DPO_PROTO_IP6]  = nbr_ip6_nodes,
    [DPO_PROTO_MPLS] = nbr_mpls_nodes,
    [DPO_PROTO_ETHERNET] = nbr_ethernet_nodes,
};

void
adj_nbr_module_init (void)
{
    dpo_register(DPO_ADJACENCY, &adj_nbr_dpo_vft, nbr_nodes);
}
```

```
void
dpo_register (dpo_type_t type, const dpo_vft_t *vft, const char * const * const * nodes)
{
    // nodes为存储协议和处理node的对应关系，参考例子

    // 边界校验：dpo_vfts vector最大index为type
    vec_validate(dpo_vfts, type);
    // 给type类型赋vft
    dpo_vfts[type] = *vft;
    /* A function to get the next VLIB node given an instance
     * of the DPO. If this is null, then the node's name MUST be
     * retreiveable from the nodes names array passed in the register
     * function
     * /
    if (NULL == dpo_vfts[type].dv_get_next_node)
    {
        dpo_vfts[type].dv_get_next_node = dpo_default_get_next_node;
    }
    // Signal on an interposed child that the parent has changed
    if (NULL == dpo_vfts[type].dv_mk_interpose)
    {
        dpo_vfts[type].dv_mk_interpose = dpo_default_mk_interpose;
    }

    // 边界校验：dpo_nodes vector最大index为type
    vec_validate(dpo_nodes, type);
    // 给type类型赋nodes
    dpo_nodes[type] = nodes;
}

dpo_type_t
dpo_register_new_type (const dpo_vft_t *vft, const char * const * const * nodes)
{
    // The DPO type value that can be assigned to the next dynamic type registration.
    dpo_type_t type = dpo_dynamic++;

    dpo_register(type, vft, nodes);

    return (type);
}
```

