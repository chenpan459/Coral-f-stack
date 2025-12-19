# F-Stack DPDK 多进程访问网络实现机制

## 概述

F-Stack 1.21.6 通过 DPDK 的多进程（Multi-Process）机制实现多个进程共享访问同一网络接口。核心原理是基于 DPDK 的共享内存（Hugepage）和进程间通信机制（Ring Buffer）来实现数据包在多进程间的分发。

## 核心机制

### 1. DPDK 多进程模型

F-Stack 使用 DPDK 的 **Primary/Secondary** 进程模型：

- **Primary 进程**（主进程）：
  - 负责初始化 DPDK 环境
  - 创建并管理共享内存区域
  - 直接访问网卡硬件，接收和发送数据包
  - 初始化网络端口配置（RSS、队列等）
  - 创建共享的数据结构和 Ring Buffer

- **Secondary 进程**（次进程）：
  - 不能初始化共享内存，只能附加到已存在的共享内存
  - 不能直接访问网卡硬件
  - 通过共享的 Ring Buffer 接收数据包
  - 可以发送数据包（通过共享的 TX 队列或 Ring）

### 2. 共享内存机制

#### 2.1 大页内存（Hugepage）

DPDK 使用大页内存来存储所有需要跨进程共享的数据结构：

```c
// 在 ff_dpdk_if.c 中，Primary 进程创建内存池
if (rte_eal_process_type() == RTE_PROC_PRIMARY) {
    pktmbuf_pool[socketid] = rte_pktmbuf_pool_create(s, nb_mbuf, ...);
} else {
    // Secondary 进程查找已存在的内存池
    pktmbuf_pool[socketid] = rte_mempool_lookup(s);
}
```

**关键点**：
- Primary 进程通过 `rte_ring_create()` 创建共享 Ring
- Secondary 进程通过 `rte_ring_lookup()` 查找并附加到共享 Ring
- 所有进程的内存映射地址相同，指针可以直接共享

#### 2.2 内存映射文件

DPDK EAL 在初始化时会：
1. Primary 进程将内存配置信息写入内存映射文件
2. Secondary 进程读取这些文件，重现相同的内存布局
3. 确保所有进程的内存映射地址一致

### 3. 数据包分发机制

#### 3.1 Dispatch Ring（分发环）

F-Stack 为每个端口和队列创建了 Dispatch Ring，用于在进程间分发数据包：

```c
// init_dispatch_ring() 创建分发环
for(queueid = 0; queueid < nb_queues; ++queueid) {
    snprintf(name_buf, RTE_RING_NAMESIZE, "dispatch_ring_p%d_q%d", portid, queueid);
    dispatch_ring[portid][queueid] = create_ring(name_buf, DISPATCH_RING_SIZE, socketid, RING_F_SC_DEQ);
}
```

**命名规则**：`dispatch_ring_p{port_id}_q{queue_id}`

#### 3.2 数据包接收流程

```
┌─────────────────────────────────────────────────────────┐
│                 网卡硬件接收数据包                        │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│     Primary 进程：rte_eth_rx_burst() 从网卡接收         │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│     process_packets() 处理数据包                         │
│     1. RSS 或自定义分发器决定目标队列                    │
│     2. 如果目标队列不是当前队列，放入 dispatch_ring     │
└──────────────────────┬──────────────────────────────────┘
                       │
         ┌─────────────┴─────────────┐
         │                           │
   当前队列处理             通过 rte_ring_enqueue()
         │                   分发到其他队列
         │                           │
         ▼                           ▼
┌─────────────────┐    ┌─────────────────────────────┐
│ ff_veth_input() │    │ dispatch_ring[port][queue]  │
│ 处理数据包      │    │ (共享内存 Ring Buffer)       │
└─────────────────┘    └─────────────┬───────────────┘
                                     │
                                     ▼
                          ┌──────────────────────────┐
                          │ Secondary 进程通过       │
                          │ rte_ring_dequeue_burst() │
                          │ 从 Ring 获取数据包        │
                          └──────────────────────────┘
```

#### 3.3 关键代码位置

**Primary 进程接收并分发数据包**：

```398:418:f-stack-1.21.6/lib/ff_dpdk_if.c
static struct rte_ring *
create_ring(const char *name, unsigned count, int socket_id, unsigned flags)
{
    struct rte_ring *ring;

    if (name == NULL) {
        rte_exit(EXIT_FAILURE, "create ring failed, no name!\n");
    }

    if (rte_eal_process_type() == RTE_PROC_PRIMARY) {
        ring = rte_ring_create(name, count, socket_id, flags);
    } else {
        ring = rte_ring_lookup(name);
    }

    if (ring == NULL) {
        rte_exit(EXIT_FAILURE, "create ring:%s failed!\n", name);
    }

    return ring;
}
```

**数据包分发逻辑**：

```1685:1693:f-stack-1.21.6/lib/ff_dpdk_if.c
            if (ret != queue_id) {
                ret = rte_ring_enqueue(dispatch_ring[port_id][ret], rtem);
                if (ret < 0) {
                    ff_traffic.rx_dropped += rtem->nb_segs;
                    rte_pktmbuf_free(rtem);
                }

                continue;
            }
```

**Secondary 进程从 Ring 接收数据包**：

```1766:1779:f-stack-1.21.6/lib/ff_dpdk_if.c
static inline int
process_dispatch_ring(uint16_t port_id, uint16_t queue_id,
    struct rte_mbuf **pkts_burst, const struct ff_dpdk_if_context *ctx)
{
    /* read packet from ring buf and to process */
    uint16_t nb_rb;
    nb_rb = rte_ring_dequeue_burst(dispatch_ring[port_id][queue_id],
        (void **)pkts_burst, MAX_PKT_BURST, NULL);

    if(nb_rb > 0) {
        process_packets(port_id, queue_id, pkts_burst, nb_rb, ctx, 1);
    }

    return nb_rb;
}
```

### 4. 进程初始化流程

#### 4.1 Primary 进程初始化

```c
ff_dpdk_init() {
    // 1. 初始化 DPDK EAL
    rte_eal_init(argc, argv);
    
    // 2. 创建内存池（共享内存）
    init_mem_pool();  // Primary 创建，Secondary 查找
    
    // 3. 创建分发 Ring（共享内存）
    init_dispatch_ring();  // Primary 创建，Secondary 查找
    
    // 4. 创建消息 Ring（进程间通信）
    init_msg_ring();
    
    // 5. 初始化网络端口（仅 Primary）
    if (rte_eal_process_type() == RTE_PROC_PRIMARY) {
        init_port_start();  // 配置网卡硬件
    }
}
```

#### 4.2 Secondary 进程初始化

Secondary 进程的初始化流程类似，但关键区别：

```407:411:f-stack-1.21.6/lib/ff_dpdk_if.c
    if (rte_eal_process_type() == RTE_PROC_PRIMARY) {
        ring = rte_ring_create(name, count, socket_id, flags);
    } else {
        ring = rte_ring_lookup(name);
    }
```

- 使用 `rte_ring_lookup()` 而不是 `rte_ring_create()`
- 使用 `rte_mempool_lookup()` 而不是 `rte_mempool_create()`
- 不执行硬件初始化（如 `port_flow_isolate()`）

### 5. 事件循环与数据包处理

所有进程都运行相同的事件循环，但数据包来源不同：

```2365:2390:f-stack-1.21.6/lib/ff_dpdk_if.c
            idle &= !process_dispatch_ring(port_id, queue_id, pkts_burst, ctx);

            nb_rx = rte_eth_rx_burst(port_id, queue_id, pkts_burst,
                MAX_PKT_BURST);
            if (nb_rx == 0)
                continue;

            idle = 0;

            /* Prefetch first packets */
            for (j = 0; j < PREFETCH_OFFSET && j < nb_rx; j++) {
                rte_prefetch0(rte_pktmbuf_mtod(
                        pkts_burst[j], void *));
            }

            /* Prefetch and handle already prefetched packets */
            for (j = 0; j < (nb_rx - PREFETCH_OFFSET); j++) {
                rte_prefetch0(rte_pktmbuf_mtod(pkts_burst[
                        j + PREFETCH_OFFSET], void *));
                process_packets(port_id, queue_id, &pkts_burst[j], 1, ctx, 0);
            }

            /* Handle remaining prefetched packets */
            for (; j < nb_rx; j++) {
                process_packets(port_id, queue_id, &pkts_burst[j], 1, ctx, 0);
            }
```

**处理顺序**：
1. 首先从 `dispatch_ring` 获取其他进程分发的数据包
2. 然后从网卡硬件接收数据包（Primary 进程）或继续从 Ring 获取（Secondary 进程）

### 6. 配置要求

#### 6.1 配置文件（config.ini）

```ini
[dpdk]
# 进程数量
nb_procs=4
# 进程 ID（每个进程不同，0 到 nb_procs-1）
proc_id=0
# 每个进程使用的 CPU 核心
proc_lcore=1,2,3,4
```

#### 6.2 启动参数

Primary 进程：
```bash
./app -l 0-3 -n 4 --proc-type=primary --file-prefix=f-stack
```

Secondary 进程：
```bash
./app -l 4-7 -n 4 --proc-type=secondary --file-prefix=f-stack
```

**关键参数**：
- `--proc-type`: 指定进程类型（primary/secondary）
- `--file-prefix`: 指定共享内存前缀，必须相同才能共享

### 7. 消息传递机制

除了数据包分发，F-Stack 还提供了进程间消息传递：

```478:518:f-stack-1.21.6/lib/ff_dpdk_if.c
static int
init_msg_ring(void)
{
    uint16_t i, j;
    uint16_t nb_procs = ff_global_cfg.dpdk.nb_procs;
    unsigned socketid = lcore_conf.socket_id;

    /* Create message buffer pool */
    if (rte_eal_process_type() == RTE_PROC_PRIMARY) {
        message_pool = rte_mempool_create(FF_MSG_POOL,
           MSG_RING_SIZE * 2 * nb_procs,
           MAX_MSG_BUF_SIZE, MSG_RING_SIZE / 2, 0,
           NULL, NULL, ff_msg_init, NULL,
           socketid, 0);
    } else {
        message_pool = rte_mempool_lookup(FF_MSG_POOL);
    }

    if (message_pool == NULL) {
        rte_panic("Create msg mempool failed\n");
    }

    for(i = 0; i < nb_procs; ++i) {
        snprintf(msg_ring[i].ring_name[0], RTE_RING_NAMESIZE,
            "%s%u", FF_MSG_RING_IN, i);
        msg_ring[i].ring[0] = create_ring(msg_ring[i].ring_name[0],
            MSG_RING_SIZE, socketid, RING_F_SP_ENQ | RING_F_SC_DEQ);
        if (msg_ring[i].ring[0] == NULL)
            rte_panic("create ring::%s failed!\n", msg_ring[i].ring_name[0]);

        for (j = FF_SYSCTL; j < FF_MSG_NUM; j++) {
            snprintf(msg_ring[i].ring_name[j], RTE_RING_NAMESIZE,
                "%s%u_%u", FF_MSG_RING_OUT, i, j);
            msg_ring[i].ring[j] = create_ring(msg_ring[i].ring_name[j],
                MSG_RING_SIZE, socketid, RING_F_SP_ENQ | RING_F_SC_DEQ);
            if (msg_ring[i].ring[j] == NULL)
                rte_panic("create ring::%s failed!\n", msg_ring[i].ring_name[j]);
        }
    }

    return 0;
}
```

### 8. 特殊数据包处理

#### 8.1 ARP/NDP 广播

ARP 和 NDP 数据包需要广播到所有进程：

```1698:1726:f-stack-1.21.6/lib/ff_dpdk_if.c
        if (filter == FILTER_ARP) {
            struct rte_mempool *mbuf_pool;
            struct rte_mbuf *mbuf_clone;
            if (!pkts_from_ring) {
                uint16_t j;
                for(j = 0; j < nb_queues; ++j) {
                    if(j == queue_id)
                        continue;

                    unsigned socket_id = 0;
                    if (numa_on) {
                        uint16_t lcore_id = qconf->port_cfgs[port_id].lcore_list[j];
                        socket_id = rte_lcore_to_socket_id(lcore_id);
                    }
                    mbuf_pool = pktmbuf_pool[socket_id];
                    mbuf_clone = pktmbuf_deep_clone(rtem, mbuf_pool);
                    if(mbuf_clone) {
                        int ret = rte_ring_enqueue(dispatch_ring[port_id][j],
                            mbuf_clone);
                        if (ret < 0) {
                            ff_traffic.rx_dropped += mbuf_clone->nb_segs;
                            rte_pktmbuf_free(mbuf_clone);
                        }
                    }
                }
            }
```

## 架构图

```
┌─────────────────────────────────────────────────────────────┐
│                   网卡硬件 (NIC)                             │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│              Primary 进程 (proc_id=0)                        │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  rte_eth_rx_burst() - 直接从网卡接收数据包            │  │
│  └────────────────────┬─────────────────────────────────┘  │
│                       │                                      │
│                       ▼                                      │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  process_packets() - 处理并分发数据包                │  │
│  │  - RSS 哈希决定目标队列                               │  │
│  │  - 或自定义分发器决定目标进程                          │  │
│  └──────┬───────────────────────┬───────────────────────┘  │
│         │                       │                           │
│   当前队列处理                  分发到其他队列                │
│         │                       │                           │
└─────────┼───────────────────────┼───────────────────────────┘
          │                       │
          │              ┌────────▼────────┐
          │              │ dispatch_ring[] │
          │              │ (共享内存 Ring)  │
          │              └────────┬────────┘
          │                       │
          │        ┌──────────────┼──────────────┐
          │        │              │              │
          ▼        ▼              ▼              ▼
┌──────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐
│Process 0 │  │Process 1│  │Process 2│  │Process 3│
│(Primary) │  │(Second) │  │(Second) │  │(Second) │
│          │  │         │  │         │  │         │
│处理数据包│  │从Ring   │  │从Ring   │  │从Ring   │
│          │  │获取数据 │  │获取数据 │  │获取数据 │
└──────────┘  └─────────┘  └─────────┘  └─────────┘
     │              │              │              │
     └──────────────┴──────────────┴──────────────┘
                      │
                      ▼
            ┌─────────────────┐
            │  共享内存区域    │
            │  - mbuf 池      │
            │  - Ring Buffer  │
            │  - 其他共享数据  │
            └─────────────────┘
```

## 关键特性

### 1. 零拷贝机制
- 数据包存储在共享内存中，进程间传递的是指针，无需复制数据

### 2. 锁无关设计
- `rte_ring` 使用无锁队列，高性能
- 单个生产者/单个消费者模式（RING_F_SP_ENQ | RING_F_SC_DEQ）

### 3. NUMA 感知
- 根据 NUMA 节点分配内存池
- 减少跨 NUMA 访问延迟

### 4. RSS 负载均衡
- 硬件 RSS 将数据包分发到不同队列
- 每个队列对应不同的进程

## 限制和注意事项

1. **ASLR 必须关闭**：Linux 的地址空间随机化会影响内存映射
   ```bash
   echo 0 > /proc/sys/kernel/randomize_va_space
   ```

2. **Primary 进程必须首先启动**：Secondary 进程需要等待 Primary 进程初始化共享内存

3. **相同 DPDK 版本**：所有进程必须使用相同版本的 DPDK

4. **文件前缀必须相同**：`--file-prefix` 参数必须一致

5. **网卡独占访问**：一个网卡只能由一个 Primary 进程访问

## 性能优化建议

1. **合理配置队列数**：队列数应该等于或大于进程数
2. **NUMA 绑定**：将进程绑定到对应的 NUMA 节点
3. **CPU 亲和性**：避免进程迁移导致的缓存失效
4. **Ring 大小**：根据数据包速率调整 `DISPATCH_RING_SIZE`

## 总结

F-Stack 通过 DPDK 的多进程机制实现了高效的网络访问共享：

1. **共享内存**：使用大页内存存储所有共享数据结构
2. **Ring Buffer**：通过无锁队列实现高效的数据包分发
3. **进程角色**：Primary 进程负责硬件访问，Secondary 进程通过 Ring 接收数据包
4. **零拷贝**：进程间传递指针而非数据，减少内存拷贝开销

这种设计使得 F-Stack 可以充分利用多核 CPU，实现高并发的网络处理能力。

