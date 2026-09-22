---
kind: error_handling
name: XCPlite 错误处理：XCP 协议级 CRC 码 + C 层 goto 错误传播与断言
category: error_handling
scope:
    - '**'
source_files:
    - src/xcp.h
    - src/xcplite.c
    - src/xcplite.h
    - inc/xcplib.h
    - src/platform.h
    - src/dbg_print.h
---

## 1. 使用的系统/方法

XCPlite 在两个层面处理错误：

- **协议层（XCP 协议）**：遵循 ASAM XCP 标准，使用 `xcp.h` 中定义的 `CRC_*` 常量（如 `CRC_CMD_OK`、`CRC_CMD_SYNTAX`、`CRC_OUT_OF_RANGE`、`CRC_ACCESS_DENIED`、`CRC_MEMORY_OVERFLOW`、`CRC_VERIFY` 等）作为命令返回值。这些值直接编码进 XCP 的 PID_ERR 响应包，由对端工具（CANape、xcpclient）解析。
- **C 运行时层**：采用“函数返回 `uint8_t` 错误码 + 局部变量 `err` + `goto negative_response`”模式，配合本地宏 `error(e)` 和 `check_error(e)` 统一跳转到负响应构造逻辑。不依赖异常机制（无 `throw/catch`），也不使用全局 `errno` 或 `perror`。

日志通过 `dbg_print.h` 中的 `DBG_PRINTF_ERROR/WARNING/INFO/DEBUG/TRACE` 宏输出，由 `XcpSetLogLevel()` 控制级别（0=关闭，4=trace）。没有独立的错误类型体系或错误对象。

## 2. 关键文件与位置

| 文件 | 作用 |
|---|---|
| `src/xcp.h` | 定义所有 XCP 命令码、PID、事件码、服务请求码以及 `CRC_*` 错误码常量 |
| `src/xcplite.c` | 协议栈核心实现；集中定义 `error(e)` / `check_error(e)` 宏，并在每个命令处理函数中用 `goto negative_response` 统一构造 CRM 响应 |
| `inc/xcplib.h` | 公共 C API；重复声明 `CRC_*` 常量供应用侧使用，并暴露 `XcpSetLogLevel()` |
| `src/platform.h` | 平台抽象层；提供 `assert` 行为、线程/互斥量封装，FreeRTOS 路径下创建任务失败时 `assert(res == pdPASS)` |
| `src/xcplite.h` | 内部接口；声明 `XcpCommand()`、`XcpBackgroundTasks()`、`XcpDisconnect()` 等状态查询函数 |
| `src/a2l.c` / `src/cal.c` / `src/shm.c` / `src/socket_raw*.c` | 各子系统按相同约定返回 `CRC_*` 码或 `bool`/`int` 错误码 |

## 3. 架构与约定

### 3.1 XCP 命令的错误传播链

`xcplite.c` 中每个命令处理函数遵循固定模式：

```c
static uint8_t XxxCommand(...) {
    uint8_t err = 0;
    // ...
    if (条件非法) error(CRC_OUT_OF_RANGE);
    check_error(XcpReadMta(...));
    // ...
negative_response:
    return err;  // 0 = CRC_CMD_OK, 非零 = CRC_* 错误码
}
```

- `error(e)` 设置 `err = e` 并 `goto negative_response`。
- `check_error(e)` 检查返回值非 0 后跳转。
- 所有子函数（`XcpReadMta`、`XcpWriteMta`、`XcpAllocDaq`、`XcpCheckMemory` 等）都返回 `CRC_*` 码，上层透传。

### 3.2 资源/内存错误的统一处理

- DAQ 内存溢出：`XcpCheckMemory()` 计算已用字节数，超过 `XCP_DAQ_MEM_SIZE` 时调用 `XcpClearDaq()` 清空配置并返回 `CRC_MEMORY_OVERFLOW`（符合 XCP 1.4 §4.1.6）。
- 事件列表溢出：`XcpCreateIndexedEvent()` 在 `e >= XCP_MAX_EVENT_COUNT` 时记录 `DBG_PRINT_ERROR("too many events")` 并返回 `XCP_UNDEFINED_EVENT_ID`。
- 名称过长：`STRNLEN(name, XCP_MAX_EVENT_NAME+1) > XCP_MAX_EVENT_NAME` 时记录错误并返回 `XCP_UNDEFINED_EVENT_ID`。

### 3.3 初始化/配置期错误

编译期通过 `#error` 强制约束：
- `XCPTL_MAX_CTO_SIZE` 必须在 8..255 之间。
- `XCPTL_MAX_DTO_SIZE < XCPTL_MAX_SEGMENT_SIZE - XCPTL_HEADER_SIZE`。
- `XCP_EPK_MAX_LENGTH` 必须为奇数且 `<128`。
- 必须定义 `XCPLITE_CONFIGURATION`，否则 `#error`。
- 必须选择一种地址模式（`ACSDD/CASDD/CXSDD/AXSDD`）。

运行期通过 `assert()` 校验前置条件：
- `XcpSetProjectName` / `XcpSetEpk` 要求参数非 NULL。
- `XcpGetProjectName` / `XcpGetLocalEpk` 在未设置时触发 `assert(0 && "... not set")`。
- DAQ 结构体大小与对齐通过 `static_assert` 与 `assert((uintptr_t)&... % 4 == 0)` 验证。

### 3.4 平台/线程创建错误

`platform.h` 中 `create_thread` 在不同平台有不同语义：
- POSIX：返回 `pthread_create` 结果（0=成功）。
- Windows：包装成 0=成功。
- FreeRTOS：是语句而非表达式，失败时 `assert(res == pdPASS)` 或 `assert(*(h) != NULL)`，无法被 `if` 捕获——这是刻意设计，使创建失败在开发期崩溃而非静默失败。

### 3.5 日志与可观测性

- `XcpSetLogLevel(level)` 设置全局日志级别（0~5），受 `OPTION_FIXED_DBG_LEVEL` 与 `OPTION_MAX_DBG_LEVEL` 限制。
- 调试输出通过 `DBG_PRINTF_ERROR/WARNING/INFO/DEBUG/TRACE` 宏，仅在启用 `OPTION_ENABLE_DBG_PRINTS` 时编译。
- 没有结构化错误码到字符串的映射表；错误信息以文本形式嵌入日志。

## 4. 约定与约束

1. **协议错误必须使用 `CRC_*` 常量**：所有 XCP 命令处理器返回值必须是 `xcp.h` 中定义的 `CRC_CMD_OK`、`CRC_CMD_SYNTAX`、`CRC_OUT_OF_RANGE`、`CRC_ACCESS_DENIED`、`CRC_MEMORY_OVERFLOW`、`CRC_VERIFY` 等之一，不得自定义数值。
2. **禁止使用 C++ 异常**：整个库是纯 C 实现，未包含 `<exception>`，也未使用 `throw/catch`；测试代码仅使用 `assert()` 做断言。
3. **禁止使用 `errno`/`perror`**：库内未出现 `errno`、`perror`、`strerror` 等 POSIX 错误通道；底层 I/O 错误通过返回 `CRC_ACCESS_DENIED` 或 `false` 表达。
4. **错误传播必须经 `error()` / `check_error()` 宏**：`xcplite.c` 中命令处理函数统一通过这两个宏跳转到 `negative_response` 标签，避免遗漏错误分支。
5. **初始化失败走 `assert` 而非返回码**：项目认为初始化阶段（`XcpInit`、`XcpSetProjectName`、`XcpSetEpk`）的错误属于编程错误，应终止进程而非恢复。
6. **DAQ 内存溢出必须清空配置**：`XcpCheckMemory` 在溢出时调用 `XcpClearDaq()` 重置所有 DAQ 状态，防止部分初始化的脏状态继续生效。
7. **FreeRTOS 任务创建失败不可检测**：`create_thread` 在 FreeRTOS 下是语句，失败即 `assert` 崩溃，调用方不能检查返回值——这是跨平台一致性的有意取舍。
8. **日志级别受构建期限制**：当 `OPTION_FIXED_DBG_LEVEL` 或 `OPTION_MAX_DBG_LEVEL` 启用时，`XcpSetLogLevel` 会拒绝超出范围的级别并记录警告。
9. **A2L/EPK 名称长度编译期约束**：`XCP_EPK_MAX_LENGTH` 与 `XCP_MAX_EVENT_NAME` 必须为奇数且 `<128`，由 `#error` 在编译期强制执行。
10. **回调钩子允许拒绝连接**：`ApplXcpRegisterConnectCallback` 允许应用返回 `false` 拒绝 CONNECT，这是唯一一处将“业务拒绝”与“协议错误”区分开的设计点。