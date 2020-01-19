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

