## 策略路由

vpp有两种方法可以实现ACL：see https://www.mail-archive.com/vpp-dev@lists.fd.io/msg02885.html

* classifier-based ACLs

```
It is faster than acl plugin, and allows only stateless operation which is 
essentially bitmask-based.
```

* acl plugin

```
Supports higher-level semantics, can parse through IPv6 extension headers, 
allows for a lightweight session tracking by specifying "permit+reflect" as 
action, but is slower than the classified based operation.

The ACL plugin is configured via API-only, there are only show commands in the 
debug CLI aimed for debugging/troubleshooting.
```

策略路由在vpp中使用abf插件实现，see https://blog.csdn.net/weixin_40815457/article/details/86523457

### 配置

### 处理