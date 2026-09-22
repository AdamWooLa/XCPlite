---
kind: configuration_system
name: XCPlite 编译期配置系统：分层 OPTION_* 宏 + CMake 配置选择 + 应用覆盖头
category: configuration_system
scope:
    - '**'
source_files:
    - src/xcplib_cfg.h
    - src/xcp_cfg.h
    - src/xcptl_cfg.h
    - src/xcplib_no_a2l_cfg.h
    - src/xcplib_ptp_cfg.h
    - src/xcplib_shm_cfg.h
    - src/xcplib_rtos_cfg.h
    - src/xcplib_raw_cfg.h
    - examples/fetchcontent_example/config/xcplib_app_cfg.h
    - CMakeLists.txt
    - cmake/xcpliteConfig.cmake.in
---

## 1. 总体方案

XCPlite 采用 **纯编译期宏配置** 的嵌入式风格配置系统，没有运行时配置文件、环境变量或 YAML/TOML。所有功能开关通过 `OPTION_*` 宏在预处理器阶段决定代码路径与内存布局，并通过 CMake 将选定的“配置覆盖头”注入到库与应用中。

核心思想是三层叠加：
- **默认层**：`src/xcplib_cfg.h` 提供全部 `OPTION_*` 的默认值（日志、时钟、DAQ、校准段、A2L、队列等）。
- **内置配置覆盖层**：`src/xcplib_<name>_cfg.h`（`no_a2l` / `ptp` / `shm` / `rtos` / `raw`）对默认值做 `#undef`/`#define` 覆盖，形成可复用的目标平台/传输模式预设。
- **应用覆盖层**：用户通过 CMake 变量 `XCPLITE_CFG_OVERRIDE` 指定自己的 `xcplib_app_cfg.h`，该文件在 `xcplib_cfg.h` 末尾被 `#include XCPLIB_CFG_OVERRIDE` 引入，用于最终定制。

CMake 在顶层 `CMakeLists.txt` 中根据 `XCPLITE_CONFIGURATION` 缓存变量选择内置覆盖头，并把它作为 `PUBLIC` compile definition `XCPLIB_CFG_OVERRIDE="..."` 传给 `xcplite` target；同时在 include path 中加入自定义覆盖头目录，使库和应用以相同方式看到它。

## 2. 关键文件

| 文件 | 作用 |
|---|---|
| `src/xcplib_cfg.h` | 全局默认配置，定义所有 `OPTION_*` 宏及版本、日志、时钟、DAQ、校准段、A2L、队列等选项 |
| `src/xcp_cfg.h` | XCP 协议层参数，依赖 `OPTION_*` 推导 `XCP_*` 常量（地址扩展、事件数、校验和、PTP、DAQ 表大小等） |
| `src/xcptl_cfg.h` | 传输层参数，依赖 `OPTION_*` 推导 `XCPTL_*` 常量（MTU→segment size、CTO/DTO 大小、队列 headroom、UDP/TCP/Raw 互斥约束） |
| `src/xcplib_no_a2l_cfg.h` | 无在线 A2L 生成配置覆盖 |
| `src/xcplib_ptp_cfg.h` | PTP 硬件时间戳配置覆盖 |
| `src/xcplib_shm_cfg.h` | 多进程共享内存模式（`OPTION_SHM_MODE`）配置覆盖 |
| `src/xcplib_rtos_cfg.h` | FreeRTOS 嵌入式配置覆盖（32 位队列、1us 时钟、无文件系统、绝对寻址） |
| `src/xcplib_raw_cfg.h` | Raw Ethernet 传输配置覆盖 |
| `examples/fetchcontent_example/config/xcplib_app_cfg.h` | 应用级覆盖示例，展示如何仅 `#undef`/`#define OPTION_*` 来裁剪功能 |
| `CMakeLists.txt` | 解析 `XCPLITE_CONFIGURATION`、校验取值、选择覆盖头、设置 `XCPLIB_CFG_OVERRIDE` 编译定义、暴露 `XCPLITE_BUILD_*` 选项 |
| `cmake/xcpliteConfig.cmake.in` | 安装后供 `find_package(xcplite)` 使用的包配置模板 |

## 3. 架构与设计决策

### 3.1 分层宏体系
- `xcplib_cfg.h` 只定义 `OPTION_*`；`xcp_cfg.h` 和 `xcptl_cfg.h` 分别 `#include xcplib_cfg.h`，再据此计算 `XCP_*` / `XCPTL_*` 常量。这样新增一个 `OPTION_*` 只需在默认层声明，协议层和传输层按需消费。
- 各层之间用 `#if defined(OPTION_*)` 守卫，未定义的选项不会触发编译错误（除了少数强制要求如 `XCPTL_MAX_SEGMENT_SIZE` 必须显式定义），便于最小化裁剪。

### 3.2 配置选择机制
- 通过 CMake 缓存变量 `XCPLITE_CONFIGURATION` 选择内置配置：`default` / `no_a2l` / `ptp` / `shm` / `rtos` / `raw`，并在 configure 阶段用 `message(FATAL_ERROR ...)` 拒绝非法值。
- 对于 `default` 配置，允许额外传入 `XCPLITE_CFG_OVERRIDE` 指向用户自定义头；若同时设置了非 default 的 `XCPLITE_CONFIGURATION`，CMake 会报错，强制“内置配置 + 应用覆盖”二选一。
- 自定义覆盖头文件名不得与内置头同名（正则匹配 `^xcplib_(no_a2l_|ptp_|shm_|rtos_|raw_)?cfg\.h$`），避免被源码树中的内置头优先找到。

### 3.3 构建产物隔离
- 注释明确“配置是互斥的 — 每个配置使用单独的 build 目录”，因为同一二进制不可能同时包含 RTOS 与 SHM 两种模式的 `OPTION_*`。
- 不同配置下启用不同的 example/test/tool target（例如 `ptp` 才构建 `ptptool`，`shm` 才构建 `shmtool`/`xcpdaemon`，`raw` 才构建 `udp_raw_demo`）。

### 3.4 运行时 vs 编译时
- 所有配置均为编译期确定；唯一可在运行时调整的是日志级别（`OPTION_DEFAULT_DBG_LEVEL` / `OPTION_FIXED_DBG_LEVEL` 注释说明固定级别不可变）。
- 没有 `.env`、`.yaml`、`.toml`、`application.properties` 等运行时配置文件；A2L/BIN/HEX 由工具链生成，不是运行期加载的配置。

## 4. 约定与约束

### 4.1 必须遵守的编译期约束（由 `#error` 强制）
- `XCPTL_MAX_CTO_SIZE` 必须是 8 的倍数且 ≤ `XCPTL_MAX_DTO_SIZE`（`xcptl_cfg.h`）。
- `OPTION_ENABLE_UDP_RAW` 与 `OPTION_ENABLE_UDP` / `OPTION_ENABLE_TCP` 互斥（`xcptl_cfg.h`）。
- `OPTION_ENABLE_UDP_RAW` 要求 `OPTION_QUEUE_32`，且不支持 SHM 模式（`xcptl_cfg.h`）。
- `XCPTL_TX_HEADROOM > 0` 时必须使用 `OPTION_QUEUE_32`（只有分段累积队列保留 segment headroom）。
- `OPTION_DAQ_EVENT_COUNT > 1000` 会触发警告（需要 >20KB 内存）。
- `CLOCK_TICKS_PER_S` 必须在 16 位可表示范围内，否则 `xcp_cfg.h` 报 `#error`。
- `XCP_MAX_EVENT_COUNT` 不能超过动态寻址的事件位宽（`XCP_DYN_ADDR_EVENT_BITS = 10`）。
- `XCPLITE_CONFIGURATION` 必须是 `default|no_a2l|ptp|shm|rtos|raw` 之一，否则 CMake 直接失败。
- 自定义 `XCPLITE_CFG_OVERRIDE` 不能命名为内置覆盖头名，否则 CMake 拒绝。

### 4.2 约定性实践
- 应用覆盖头应只使用 `#undef`/`#define OPTION_*`，不修改其他宏（见 `examples/fetchcontent_example/config/xcplib_app_cfg.h` 的注释与正文）。
- 如需基于某个内置配置裁剪，应在自定义覆盖头顶部 `#include "xcplib_<name>_cfg.h"` 再覆盖（CMake 注释已说明此用法）。
- 每个 `OPTION_*` 都有对应文档位置：`docs/xcplib_cfg.md` 是配置参考，`docs/SOCKET_RAW.md` 描述 raw 传输限制，`docs/BUILDING.md` 描述构建选项。
- 配置变更必须配合独立 build 目录，因为 `OPTION_*` 会改变内存布局（DAQ 表大小、队列缓冲、校准段池等）。

### 4.3 对外暴露点
- 通过 `target_compile_definitions(xcplite PUBLIC "XCPLITE_CONFIGURATION=\"${XCPLITE_CONFIGURATION}\"")` 把当前配置名暴露给下游，便于条件编译。
- 安装时将选中的覆盖头复制到 `include/`，使 `find_package(xcplite)` 的消费者能知道库是以哪个配置构建的。
