## 邻接(Adjacency)

临接关系是附加的L3对等方的表示(An adjacency is a representation of an attached L3 peer)。

三种临接子类型(Adjacency Sub-types)：
* **邻居(neighbor)**：附加的L3对等方的表示(a representation of an attached L3 peer)。
  - 键(Key):{addr, interface, link/ether-type}
  - 共享的(SHARED)
* **收集(glean)**：用于驱动发往本地子网的数据包的ARP/ND(used to drive ARP/ND for packets destined to a local sub-net)。
  - “glean”表示使用数据包的目的地址作为目标('glean' mean use the packet's destination address as the target)
  - 非共享的(UNSHARED)，每个接口一个(only one per-interface)。
* **中链(midchain)**：虚拟/隧道接口上的邻接关系(a neighbor adj on a virtual/tunnel interface)。

邻接节点的下一个节点，公共枚举类型：
```
/** @brief Common (IP4/IP6) next index stored in adjacency. */
typedef enum
{
  /** Adjacency to drop this packet. */
  IP_LOOKUP_NEXT_DROP,
  /** Adjacency to punt this packet. */
  IP_LOOKUP_NEXT_PUNT,

  /** This packet is for one of our own IP addresses. */
  IP_LOOKUP_NEXT_LOCAL,

  /** This packet matches an "incomplete adjacency" and packets
     need to be passed to ARP to find rewrite string for
     this destination. */
  IP_LOOKUP_NEXT_ARP,

  /** This packet matches an "interface route" and packets
     need to be passed to ARP to find rewrite string for
     this destination. */
  IP_LOOKUP_NEXT_GLEAN,

  /** This packet is to be rewritten and forwarded to the next
     processing node.  This is typically the output interface but
     might be another node for further output processing. */
  IP_LOOKUP_NEXT_REWRITE,

  /** This packets follow a mid-chain adjacency */
  IP_LOOKUP_NEXT_MIDCHAIN,

  /** This packets needs to go to ICMP error */
  IP_LOOKUP_NEXT_ICMP_ERROR,

  /** Multicast Adjacency. */
  IP_LOOKUP_NEXT_MCAST,

  /** Broadcasr Adjacency. */
  IP_LOOKUP_NEXT_BCAST,

  /** Multicast Midchain Adjacency. An Adjacency for sending macst packets
   *  on a tunnel/virtual interface */
  IP_LOOKUP_NEXT_MCAST_MIDCHAIN,

  IP_LOOKUP_N_NEXT,
} __attribute__ ((packed)) ip_lookup_next_t;
```

邻接节点的下一个节点，IPv4枚举类型：
```
typedef enum
{
  IP4_LOOKUP_N_NEXT = IP_LOOKUP_N_NEXT,
} ip4_lookup_next_t;
```

邻接节点的下一个节点，IPv6枚举类型：
```
typedef enum
{
  /* Hop-by-hop header handling */
  IP6_LOOKUP_NEXT_HOP_BY_HOP = IP_LOOKUP_N_NEXT,
  IP6_LOOKUP_NEXT_ADD_HOP_BY_HOP,
  IP6_LOOKUP_NEXT_POP_HOP_BY_HOP,
  IP6_LOOKUP_N_NEXT,
} ip6_lookup_next_t;
```


IP单播邻接结构体：
```
/**
 * @brief IP unicast adjacency.
 *  @note cache aligned.
 *
 * An adjacency is a representation of a peer on a particular link.
 */
typedef struct ip_adjacency_t_
{
  CLIB_CACHE_LINE_ALIGN_MARK (cacheline0);

  /**
   * Linkage into the FIB node graph. First member since this type
   * has 8 byte alignment requirements.
   */
  fib_node_t ia_node;

  /**
   * Next hop after ip4-lookup.
   *  This is not accessed in the rewrite nodes.
   * 1-bytes
   */
  ip_lookup_next_t lookup_next_index;

  /**
   * link/ether-type
   * 1 bytes
   */
  vnet_link_t ia_link;

  /**
   * The protocol of the neighbor/peer. i.e. the protocol with
   * which to interpret the 'next-hop' attributes of the sub-types.
   * 1-btyes
   */
  fib_protocol_t ia_nh_proto;

  /**
   * Flags on the adjacency
   * 1-bytes
   */
  adj_flags_t ia_flags;

  union
  {
    /**
     * IP_LOOKUP_NEXT_ARP/IP_LOOKUP_NEXT_REWRITE
     *
     * neighbour adjacency sub-type;
     */
    struct
    {
      ip46_address_t next_hop;
    } nbr;
      /**
       * IP_LOOKUP_NEXT_MIDCHAIN
       *
       * A nbr adj that is also recursive. Think tunnels.
       * A nbr adj can transition to be of type MDICHAIN
       * so be sure to leave the two structs with the next_hop
       * fields aligned.
       */
    struct
    {
      /**
       * The recursive next-hop.
       *  This field MUST be at the same memory location as
       *   sub_type.nbr.next_hop
       */
      ip46_address_t next_hop;
      /**
       * The next DPO to use
       */
      dpo_id_t next_dpo;
      /**
       * A function to perform the post-rewrite fixup
       */
      adj_midchain_fixup_t fixup_func;
      /**
       * Fixup data passed back to the client in the fixup function
       */
      const void *fixup_data;
      /**
       * the FIB entry this midchain resolves through. required for recursive
       * loop detection.
       */
      fib_node_index_t fei;
    } midchain;
    /**
     * IP_LOOKUP_NEXT_GLEAN
     *
     * Glean the address to ARP for from the packet's destination.
     * Technically these aren't adjacencies, i.e. they are not a
     * representation of a peer. One day we might untangle this coupling
     * and use a new Glean DPO.
     */
    struct
    {
      ip46_address_t receive_addr;
    } glean;
  } sub_type;

  CLIB_CACHE_LINE_ALIGN_MARK (cacheline1);

  /* Rewrite in second/third cache lines */
  VNET_DECLARE_REWRITE;

  /**
   * more control plane members that do not fit on the first cacheline
   */
  /**
   * A sorted vector of delegates
   */
  struct adj_delegate_t_ *ia_delegates;

  /**
   * The VLIB node in which this adj is used to forward packets
   */
  u32 ia_node_index;
} ip_adjacency_t;
```