# F-Stack Redis 5.0.5 代码结构和使用方法

## 概述

这是基于 Redis 5.0.5 版本的 F-Stack 集成版本。F-Stack 是一个高性能的网络框架，基于 DPDK 和 FreeBSD 内核协议栈实现。该 Redis 版本通过 F-Stack 提供了更高的网络性能。

## 代码结构

### 1. 根目录结构

```
redis-5.0.5/
├── src/              # Redis 核心源代码
├── deps/             # 依赖库（hiredis, jemalloc, lua, linenoise）
├── tests/            # 测试文件
├── utils/            # 工具脚本
├── Makefile          # 顶层 Makefile
├── redis.conf        # Redis 配置文件
├── sentinel.conf     # Sentinel 配置文件
└── README.md         # 说明文档
```

### 2. 核心源代码目录 (src/)

#### 2.1 F-Stack 集成相关文件

- **ae_ff_kqueue.c**: F-Stack 事件循环实现，基于 kqueue
  - 实现了 Redis 事件循环的 F-Stack 版本
  - 使用 `ff_kqueue()` 和 `ff_kevent()` 替代标准系统调用

- **anet_ff.c / anet_ff.h**: F-Stack 网络接口封装
  - 封装了 socket、bind、connect、accept、send、recv 等系统调用
  - 通过 `ff_fdisused()` 判断文件描述符是否为 F-Stack 文件描述符
  - 自动将标准系统调用路由到 F-Stack API

#### 2.2 主要核心文件

- **server.c / server.h**: Redis 服务器主程序
  - `main()` 函数入口
  - 在 `HAVE_FF_KQUEUE` 宏定义时，初始化 F-Stack：
    ```c
    ff_init(argc, argv);  // 初始化 F-Stack
    ff_mod_init();        // 初始化网络模块
    ```
  - 事件循环通过 `ff_run(loop, server.el)` 启动

- **ae.c / ae.h**: 事件循环抽象层
  - 根据编译选项选择不同的事件驱动实现
  - 优先级：`HAVE_FF_KQUEUE` > `HAVE_EVPORT` > `HAVE_EPOLL` > `HAVE_KQUEUE` > `select`

- **networking.c**: 网络 I/O 处理
- **db.c**: 数据库操作
- **object.c**: Redis 对象实现
- **replication.c**: 主从复制
- **cluster.c**: 集群功能

#### 2.3 数据类型实现文件

- `t_string.c`: 字符串类型
- `t_list.c`: 列表类型
- `t_hash.c`: 哈希类型
- `t_set.c`: 集合类型
- `t_zset.c`: 有序集合类型
- `t_stream.c`: 流类型（Redis 5.0 新特性）

### 3. 依赖库 (deps/)

- **hiredis**: Redis C 客户端库
- **jemalloc**: 内存分配器（Linux 默认）
- **lua**: Lua 脚本支持
- **linenoise**: 命令行编辑库

## F-Stack 集成机制

### 1. 编译时配置

在 `src/Makefile` 中：

```makefile
# F-Stack 路径配置
FF_PATH=/usr/local          # F-Stack 安装路径
FF_DPDK=/usr/local          # DPDK 安装路径

# 编译选项
FINAL_CFLAGS += -DHAVE_FF_KQUEUE
FINAL_CFLAGS += -I$(FF_PATH)/lib

# 链接选项
FINAL_LIBS += -L${FF_PATH}/lib -Wl,--whole-archive,-lfstack,--no-whole-archive
FINAL_LIBS += -L${FF_DPDK}/lib -Wl,--whole-archive,-ldpdk,--no-whole-archive
```

### 2. 系统调用拦截

`anet_ff.c` 实现了系统调用的动态拦截：

- 使用 `dlsym(RTLD_NEXT, ...)` 获取原始系统调用
- 通过 `ff_fdisused()` 判断是否为 F-Stack 文件描述符
- 如果是，则路由到 F-Stack API（如 `ff_socket()`, `ff_bind()` 等）
- 否则使用原始系统调用

### 3. 事件循环替换

- 标准 Redis 使用 `aeMain()` 运行事件循环
- F-Stack 版本使用 `ff_run(loop, server.el)` 运行事件循环
- 事件轮询通过 `ff_kevent()` 实现

## 编译方法

### 1. 前置条件

- 已安装 F-Stack（默认路径 `/usr/local`）
- 已安装 DPDK（默认路径 `/usr/local`）
- 编译工具：gcc、make

### 2. 环境变量配置（可选）

```bash
export FF_PATH=/path/to/f-stack    # 如果 F-Stack 不在默认路径
export FF_DPDK=/path/to/dpdk       # 如果 DPDK 不在默认路径
```

### 3. 编译步骤

```bash
# 进入 Redis 源码目录
cd f-stack-1.21.6/app/redis-5.0.5

# 编译
make

# 或者指定自定义路径
make FF_PATH=/custom/f-stack/path FF_DPDK=/custom/dpdk/path
```

### 4. 编译输出

编译完成后，可执行文件位于：
- `src/redis-server`: Redis 服务器
- `src/redis-cli`: Redis 客户端
- `src/redis-benchmark`: 性能测试工具
- `src/redis-check-aof`: AOF 文件检查工具
- `src/redis-check-rdb`: RDB 文件检查工具

## 运行方法

### 1. 启动 F-Stack Redis 服务器

F-Stack Redis 需要特殊的启动参数，因为需要在启动时传递 F-Stack 和 DPDK 的参数。

标准启动方式（从 F-Stack 启动脚本调用）：

```bash
# 通常通过 F-Stack 的启动脚本启动
# 启动脚本会传递必要的 F-Stack/DPDK 参数（如 -l, -n, --proc-type 等）
```

或者直接启动（需要手动传递 F-Stack 参数）：

```bash
cd src
./redis-server \
  -l 0-3 \                    # CPU 核心列表
  -n 4 \                      # 内存通道数
  --proc-type=primary \       # 进程类型
  --file-prefix=redis \       # 共享内存前缀
  /path/to/redis.conf         # 配置文件路径
```

**注意**: F-Stack 参数会被 `ff_init()` 处理，然后从 argv 中移除（见 `server.c:4056-4068`）

### 2. 配置文件

使用标准的 `redis.conf` 配置文件，但需要注意：

- `bind`: 绑定地址（例如 `bind 0.0.0.0`）
- `port`: 监听端口（默认 6379）
- `daemonize`: 是否后台运行

### 3. 使用 Redis CLI 连接

```bash
cd src
./redis-cli -h <server_ip> -p 6379
```

### 4. 性能测试

```bash
cd src
./redis-benchmark -h <server_ip> -p 6379 -n 100000 -c 50
```

## 关键代码位置

### 1. F-Stack 初始化

```12:14:f-stack-1.21.6/app/redis-5.0.5/src/server.c
#ifdef HAVE_FF_KQUEUE
    int rc = ff_init(argc, argv);
    assert(0 == rc);
    ff_mod_init();
```

### 2. 事件循环启动

```4260:4260:f-stack-1.21.6/app/redis-5.0.5/src/server.c
    ff_run(loop, server.el);
```

### 3. 事件循环实现

```49:50:f-stack-1.21.6/app/redis-5.0.5/src/ae.c
 #ifdef HAVE_FF_KQUEUE
#include "ae_ff_kqueue.c"
```

### 4. 网络接口拦截

网络系统调用的拦截实现在 `anet_ff.c` 中，例如 socket 函数：

```154:171:f-stack-1.21.6/app/redis-5.0.5/src/anet_ff.c
socket(int domain, int type, int protocol)
{
    int rc;

    if (unlikely(inited == 0)) {
        INIT_FUNCTION(socket);
        return real_socket(domain, type, protocol);
    }

    if ((AF_INET != domain) || (SOCK_STREAM != type && SOCK_DGRAM != type)) {
        rc = real_socket(domain, type, protocol);
        return rc;
    }

    rc = ff_socket(domain, type, protocol);

    return rc;
}
```

## 与标准 Redis 的区别

1. **事件循环**: 使用 F-Stack 的 kqueue 实现，而不是 epoll/kqueue
2. **网络 I/O**: 系统调用被拦截并路由到 F-Stack API
3. **启动参数**: 需要传递 F-Stack/DPDK 参数
4. **性能**: 利用 DPDK 用户态协议栈，获得更高的网络性能

## 注意事项

1. **权限要求**: DPDK 需要以 root 权限运行，或者配置 hugepages
2. **CPU 亲和性**: 会自动设置 CPU 亲和性（见 `server.c:2771-2789`）
3. **内存要求**: DPDK 需要大页内存支持
4. **网络配置**: 需要绑定物理网卡到 DPDK 驱动

## 故障排查

1. **编译错误**: 检查 FF_PATH 和 FF_DPDK 环境变量
2. **运行时错误**: 检查是否有足够的大页内存
3. **网络问题**: 确认网卡已正确绑定到 DPDK

## 参考资料

- F-Stack 官方文档: `f-stack-1.21.6/doc/`
- Redis 官方文档: https://redis.io/documentation
- DPDK 文档: http://dpdk.org/doc

