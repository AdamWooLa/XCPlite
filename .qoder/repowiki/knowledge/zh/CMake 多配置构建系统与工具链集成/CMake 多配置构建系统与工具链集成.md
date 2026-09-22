---
kind: build_system
name: CMake 多配置构建系统与工具链集成
category: build_system
scope:
    - '**'
source_files:
    - CMakeLists.txt
    - cmake/xcpliteConfig.cmake.in
    - build.sh
    - build.bat
    - docs/BUILDING.md
    - src/xcplib_cfg.h
    - src/xcplib_no_a2l_cfg.h
    - src/xcplib_ptp_cfg.h
    - src/xcplib_shm_cfg.h
    - src/xcplib_rtos_cfg.h
    - src/xcplib_raw_cfg.h
---

## 1. 使用的系统/方法

项目以 **CMake 3.14+** 为唯一构建系统，顶层 `CMakeLists.txt` 同时管理核心库、示例、测试、工具与安装规则。通过一个互斥的 `XCPLITE_CONFIGURATION` 变量（`default` / `no_a2l` / `ptp` / `shm` / `rtos` / `raw`）在单一源码树中编译出不同功能集的二进制产物；每个配置必须使用独立的 build 目录。

- 语言标准：C11（强制）、C++17（非 Windows）/ C++20（Windows），均关闭扩展。
- 编译器探测：GCC / Clang / MSVC 分别设置警告标志和 Release/RelWithDebInfo/Debug 优化级别（Release 默认 `-O2 -DNDEBUG`，RelWithDebInfo 默认 `-O1 -g -DNDEBUG`，Debug 默认 `-O0 -g`）。
- 依赖：`find_package(Threads REQUIRED)`，Unix 下链接 `m`，可选链接 `atomic`（ARM Clang 64-bit atomics）。
- Rust 工具（`xcpclient`、`bintool`）通过 `add_custom_target` + `cargo` 调用集成到同一 CMake 流程中。
- 安装：生成 `xcpliteTargets.cmake` 与 `xcpliteConfig.cmake`（含 `SameMajorVersion` 版本兼容），导出命名空间 `xcplite::xcplite`，供外部 `find_package(xcplite)` 消费。

## 2. 关键文件

- `CMakeLists.txt` — 全部构建逻辑：配置选择、源文件列表、目标注册、选项开关、安装规则。
- `cmake/xcpliteConfig.cmake.in` — CMake 包配置文件模板，注入已安装的 `XCPLITE_CONFIGURATION` 并声明依赖。
- `build.sh` — 用户入口脚本，封装 CMake 配置、构建、安装、clang-tidy、cargo install，按配置映射到独立 build 目录（`build` / `build-no_a2l` / `build-ptp` / `build-shm` / `build-rtos` / `build-raw`）。
- `build.bat` — Windows 下生成 Visual Studio 17 2022 解决方案。
- `docs/BUILDING.md` — 完整的构建文档，列出所有配置、选项、目标矩阵与故障排查。
- `src/xcplib_cfg.h` + `src/xcplib_no_a2l_cfg.h` / `xcplib_ptp_cfg.h` / `xcplib_shm_cfg.h` / `xcplib_rtos_cfg.h` / `xcplib_raw_cfg.h` — 可插拔的配置覆盖头，被 `xcplib_cfg.h` 末尾包含，控制 `OPTION_*` 宏。
- `examples/*/CMakeLists.txt` 与 `tools/*/Cargo.toml` — 独立子工程，演示通过 `FetchContent` 或 `find_package` 消费 xcplite。

## 3. 架构与约定

### 3.1 互斥配置模型
根 CMake 根据 `XCPLITE_CONFIGURATION` 决定编译哪些源文件和定义哪些宏：
- `default`：完整功能（A2L 生成、文件系统、UDP/TCP）。
- `no_a2l`：禁用 on-target A2L，A2L 由 xcpclient 从 ELF 生成。
- `ptp`：启用 socket 硬件时间戳（Linux + PTP NIC）。
- `shm`：共享内存多进程模式（配套 `shmtool`、`xcpdaemon`）。
- `rtos`：FreeRTOS 嵌入式目标，无文件系统，32 位。
- `raw`：原始以太网传输（AF_PACKET，Linux only）。

配置覆盖头通过 `target_compile_definitions(PUBLIC "XCPLIB_CFG_OVERRIDE=\"...\"")` 暴露给消费者，确保库与使用者看到相同的 `OPTION_*` 展开结果（结构体布局一致）。

### 3.2 顶层 vs 子项目行为
通过 `if(CMAKE_SOURCE_DIR STREQUAL PROJECT_SOURCE_DIR)` 判断是否顶层项目：
- 顶层：设置全局 `CMAKE_<LANG>_FLAGS_<CONFIG>`、默认 `CMAKE_INSTALL_PREFIX` 为 `<build>/install`、开启 `CMAKE_EXPORT_COMPILE_COMMANDS`、启用 `XCPLITE_INSTALL`。
- 作为 `add_subdirectory` / `FetchContent` 消费时：不污染父项目的缓存变量，默认关闭安装规则，避免 `cmake --install` 误装 xcplite。

### 3.3 目标组织
- 库目标：`xcplite` + 别名 `xcplite::xcplite`。
- 示例：按配置条件 `add_executable`，如 `hello_xcp`、`cpp_demo`、`udp_raw_demo`、`freertos_emu_demo` 等。
- 测试：`a2l_test`、`cal_test`、`daq_test`、`clock_test`、`queue_test`、`type_detection_test_c/cpp`、`socket_raw_test` 等，由 `XCPLITE_BUILD_TESTS` 控制。
- 工具：`ptptool`（ptp）、`shmtool`/`xcpdaemon`（shm），由 `XCPLITE_BUILD_TOOLS` 控制。
- Rust 工具：`xcpclient`、`bintool`，由 `XCPLITE_BUILD_RUST_TOOLS` 控制，通过 `add_custom_target` 调用 cargo。

### 3.4 安装与分发
安装产物包括：静态库、`inc/*.h` 公共头、`src/platform.h`、`src/sockets.h`、`src/socket_raw_hal.h`、当前配置的覆盖头、`lib/cmake/xcplite/xcpliteTargets.cmake`、`xcpliteConfig.cmake`、`xcpliteConfigVersion.cmake`（`COMPATIBILITY SameMajorVersion`）。`xcpliteConfig.cmake` 中记录 `XCPLITE_CONFIGURATION_INSTALLED`，供下游校验兼容性。

## 4. 约定与约束

- **每配置一 build 目录**：配置之间互斥，混用同一 build 目录会产生错误结果（CMake 文档与 `build.sh` 均强调）。
- **配置名受严格白名单限制**：`property STRINGS default no_a2l ptp shm rtos raw`，非法值触发 `FATAL_ERROR`。
- **应用级覆盖头仅对 `default` 配置有效**：若指定 `XCPLITE_CFG_OVERRIDE` 但配置不是 `default`，直接报错；且文件名不得与 shipped 头同名（会被 `src/` 优先匹配）。
- **平台相关目标有条件构建**：`ptp4l_demo`、`ptptool`、`udp_raw_demo`、`socket_raw_test`、`bpf_demo` 仅在 Linux 下构建；`shmtool`/`xcpdaemon` 跳过 Windows；`freertos_emu_demo` 跳过 Windows。
- **Rust 工具是可选的**：未找到 `cargo` 时仅发 warning 并跳过，不影响 C/C++ 构建。
- **默认不安装**：仅当 xcplite 作为顶层项目时 `XCPLITE_INSTALL=ON`；作为子项目时默认关闭，避免污染父项目安装。
- **编译器切换需显式传参**：`build.sh` 通过 `-DCMAKE_C_COMPILER=$CC` 等参数绕过 CMake 缓存，使 `CC`/`CXX` 环境变量生效。
- **版本策略**：`project(xcplite VERSION 2.2.2)`，安装时使用 `write_basic_package_version_file(... COMPATIBILITY SameMajorVersion)`，要求主版本号一致。
- **调试符号策略**：Release 去断言（`-DNDEBUG`），RelWithDebInfo 保留中等优化与调试信息（`-O1 -g`），便于栈测量；MSVC 对应 `/O2 /Zi` 与 `/Oy-` 禁用帧指针省略。
- **编译命令导出**：顶层构建默认开启 `CMAKE_EXPORT_COMPILE_COMMANDS`，配合 `.clang-tidy` 与 `build.sh tidy` 支持 clang-tidy 检查。
- **示例/测试/工具的构建完全由选项门控**：`XCPLITE_BUILD_EXAMPLES`、`XCPLITE_BUILD_TESTS`、`XCPLITE_BUILD_TOOLS`、`XCPLITE_BUILD_RUST_TOOLS`、`XCPLITE_BUILD_BPF_DEMO` 均为 OFF 默认，需显式开启。