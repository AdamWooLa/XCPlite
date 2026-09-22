---
kind: logging_system
name: XCPlite 调试日志系统：基于宏的分级 printf 输出与可配置日志级别
category: logging_system
scope:
    - '**'
source_files:
    - src/dbg_print.h
    - src/xcplib_cfg.h
    - src/xcplite.c
    - inc/xcplib.h
---

## 1. 使用的系统与方案

XCPlite 没有引入第三方日志框架，而是实现了一套**纯 C 宏 + 编译期/运行期可配置级别**的轻量级调试日志子系统。核心由 `src/dbg_print.h` 提供，通过 `OPTION_ENABLE_DBG_PRINTS`、`OPTION_DEFAULT_DBG_LEVEL`、`OPTION_MAX_DBG_LEVEL`、`OPTION_FIXED_DBG_LEVEL`、`OPTION_ENABLE_DBG_STDERR` 等编译选项控制行为。

- **日志级别**（1–6）：1=Error、2=Warn、3=Info、4=Trace（打印所有 XCP 命令）、5=Debug、6=Very verbose。
- **输出目标**：默认走 `printf`；若定义 `OPTION_ENABLE_DBG_STDERR`，则 Error/Warning 走 `fprintf(stderr, ...)`，其余仍走 `stdout`。
- **ANSI 颜色**：在非 FreeRTOS 环境下为不同级别添加 ANSI 转义码（红/黄/灰/绿/蓝/品红），FreeRTOS 下为空宏。
- **可选错误位置**：启用 `DBG_PRINT_ERROR_LOCATION` 后，ERROR 消息会包含 `__FILE__` 和 `__LINE__`。
- **运行时 vs 固定级别**：未定义 `OPTION_FIXED_DBG_LEVEL` 时，`DBG_LEVEL` 解析为全局变量 `gXcpLogLevel`（在 `src/xcplite.c` 中定义并暴露于 `inc/xcplib.h`），可通过 `XcpSetLogLevel(level)` 动态调整；定义了 `OPTION_FIXED_DBG_LEVEL` 时，级别在编译期固定，调用 `XcpSetLogLevel` 会被忽略并打印警告。
- **裁剪优化**：当 `OPTION_MAX_DBG_LEVEL < N` 时，`DBG_PRINTFN` / `DBG_PRINTN` 被重定义为空宏，编译器直接消除对应代码路径。

## 2. 关键文件与包

| 文件 | 作用 |
|---|---|
| `src/dbg_print.h` | 日志宏定义、级别判定、ANSI 颜色、stderr/stdout 选择 |
| `src/xcplib_cfg.h` | 默认日志开关与级别（`OPTION_ENABLE_DBG_PRINTS`、`OPTION_DEFAULT_DBG_LEVEL=3`、`OPTION_MAX_DBG_LEVEL=4`、`OPTION_ENABLE_DBG_STDERR`） |
| `src/xcplite.c` | 定义 `uint8_t gXcpLogLevel = OPTION_DEFAULT_DBG_LEVEL;` 及 `XcpSetLogLevel()` 实现 |
| `inc/xcplib.h` | 对外暴露 `void XcpSetLogLevel(uint8_t level);`，文档说明 level 0–5 |
| 各示例 `examples/*/src/main.c` | 统一在启动时调用 `XcpSetLogLevel(OPTION_LOG_LEVEL);` |

## 3. 架构与约定

- **单点入口**：所有模块通过 `#include "dbg_print.h"` 使用统一的 `DBG_PRINT*` 系列宏，不直接调用 `printf`/`fprintf`，从而保证级别过滤与颜色一致。
- **编译期裁剪**：`OPTION_ENABLE_DBG_PRINTS` 关闭时，所有 DBG_* 宏全部展开为空操作，零开销；`OPTION_MAX_DBG_LEVEL` 进一步在编译期移除高于阈值的级别分支。
- **运行期可调**：默认模式下 `DBG_LEVEL` 绑定到 `gXcpLogLevel`，可在程序运行期间通过 `XcpSetLogLevel()` 提升或降低输出粒度，便于调试定位。
- **应用覆盖机制**：`xcplib_cfg.h` 末尾支持通过 `XCPLIB_CFG_OVERRIDE` 包含用户自定义配置头（如 `examples/fetchcontent_example/config/xcplib_app_cfg.h`），允许下游项目在不修改库源码的情况下覆盖日志级别。
- **传输层/协议层集成**：`src/xcplite.c` 在多处使用 `DBG_PRINTF3`/`DBG_PRINTF4` 记录 XCP 命令与响应（level≥4 即 Trace），配合 `XcpSetLogLevel(4)` 可获得完整的协议交互跟踪。

## 4. 约定与约束

- **必须定义级别来源**：如果启用了 `OPTION_ENABLE_DBG_PRINTS`，则必须同时定义 `OPTION_DEFAULT_DBG_LEVEL` 或 `OPTION_FIXED_DBG_LEVEL`，否则编译报错（`#error "Please define OPTION_DEFAULT_DBG_LEVEL or OPTION_FIXED_DBG_LEVEL"`）。
- **最大级别不低于默认级别**：若未使用固定级别，`OPTION_MAX_DBG_LEVEL` 必须 ≥ `OPTION_DEFAULT_DBG_LEVEL` 且 ≥ 2，否则编译报错。
- **固定级别模式不可变**：当定义 `OPTION_FIXED_DBG_LEVEL` 时，`XcpSetLogLevel()` 不会改变实际级别，而是通过 `DBG_PRINTF_ERROR` 输出一条提示“fixed log level”，因此该模式下日志级别是编译期常量。
- **示例工程统一约定**：所有示例在初始化 XCP 后、启动服务器前调用 `XcpSetLogLevel(OPTION_LOG_LEVEL);`，将应用层定义的 `OPTION_LOG_LEVEL` 传入，保持日志级别与应用配置一致。
- **无结构化字段**：当前日志系统仅输出格式化字符串，不包含 JSON/XML 等结构化字段、时间戳、线程 ID 等元数据；如需结构化日志需在上层封装。
- **平台差异**：FreeRTOS 环境禁用 ANSI 颜色；非 FreeRTOS 环境保留颜色以增强可读性。

## 5. 总结

XCPlite 的日志系统是一个**高度可裁剪、零依赖、支持编译期与运行期两级控制的 printf 包装层**。它通过 `dbg_print.h` 提供的宏族屏蔽了级别判断、颜色注入与 stderr 分流逻辑，并通过 `xcplib_cfg.h` 中的编译选项集中管理行为，再通过 `XcpSetLogLevel()` 暴露运行期调节能力。这种设计使同一份源码能在嵌入式（关闭日志或固定低级别）与桌面调试（开启 trace 级别）之间灵活切换，而无需修改业务代码。