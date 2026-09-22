---
kind: dependency_management
name: CMake + Cargo 多语言依赖管理（xcplite 核心库与工具链）
category: dependency_management
scope:
    - '**'
source_files:
    - CMakeLists.txt
    - cmake/xcpliteConfig.cmake.in
    - src/xcplib_cfg.h
    - src/xcplib_no_a2l_cfg.h
    - src/xcplib_ptp_cfg.h
    - src/xcplib_shm_cfg.h
    - src/xcplib_rtos_cfg.h
    - src/xcplib_raw_cfg.h
    - examples/fetchcontent_example/CMakeLists.txt
    - examples/cmp_demo/CMakeLists.txt
    - examples/external_example/CMakeLists.txt
    - examples/silkit_demo/CMakeLists.txt
    - examples/freertos_demo/freertos_emu_demo/CMakeLists.txt
    - tools/xcpclient/Cargo.toml
    - tools/xcpclient/Cargo.lock
    - tools/bintool/Cargo.toml
---

## 1. 使用的系统与方式

仓库采用**双轨依赖管理**：
- C/C++ 核心库 `xcplite` 使用 **CMake**（最低版本 3.14，见根 `CMakeLists.txt` 第 2 行），通过 `find_package(Threads REQUIRED)` 等标准模块发现系统库，并通过 `check_library_exists` 探测可选的 `atomic` 库。
- Rust 工具 `xcpclient`、`bintool` 使用 **Cargo**（`tools/xcpclient/Cargo.toml`、`tools/bintool/Cargo.toml`），并随仓库提交 `Cargo.lock` 锁定所有 crate 的版本与校验和。
- 第三方 C/C++ 依赖以**源码子目录 + FetchContent** 形式引入（如 FreeRTOS-Kernel），没有 vendored 二进制或 git submodule 形式的静态依赖。
- 项目本身作为可被 `add_subdirectory` / `FetchContent` / `find_package` 消费的 CMake 包，安装时生成 `xcpliteTargets.cmake` 与 `xcpliteConfig.cmake`（根 `CMakeLists.txt` 598–649 行）。

## 2. 关键文件

| 文件 | 作用 |
|---|---|
| `CMakeLists.txt` | 顶层构建入口，声明 `XCPLITE_CONFIGURATION`、`XCPLITE_BUILD_*` 选项，定义 `xcplite` 库目标、示例、测试、工具及安装规则 |
| `cmake/xcpliteConfig.cmake.in` | CMake 包配置模板，供下游 `find_package(xcplite)` 使用 |
| `src/xcplib_cfg.h` 及 `src/xcplib_{no_a2l,ptp,shm,rtos,raw}_cfg.h` | 编译期特性开关头，通过 `XCPLITE_CFG_OVERRIDE` 机制由应用覆盖 |
| `examples/fetchcontent_example/CMakeLists.txt` | 演示用 `FetchContent_Declare(xcplite ...)` 从源码树消费本项目的完整用法 |
| `examples/cmp_demo/CMakeLists.txt`、`examples/external_example/CMakeLists.txt`、`examples/silkit_demo/CMakeLists.txt` | 演示通过 `find_package(xcplite REQUIRED)` 消费已安装的 xcplite |
| `tools/xcpclient/Cargo.toml` + `Cargo.lock` | Rust 工具依赖声明与精确锁定 |
| `tools/bintool/Cargo.toml` | 轻量 Rust 工具的依赖声明 |
| `examples/freertos_demo/freertos_emu_demo/CMakeLists.txt` | 通过 `FetchContent` 拉取 FreeRTOS-Kernel 源码 |

## 3. 架构与约定

### 3.1 C/C++ 依赖：单源码树 + 可插拔配置
- 所有 xcplite 源码集中在 `src/`，不依赖外部 C/C++ 库（除系统 `Threads`、`m`、可选 `atomic`）。平台差异通过 `#ifdef WIN32/APPLE/UNIX` 与 `XCPLITE_CONFIGURATION` 选择不同实现源文件（如 `socket_raw_hal_linux.c`）。
- 功能裁剪通过互斥的 `XCPLITE_CONFIGURATION`（`default | no_a2l | ptp | shm | rtos | raw`）切换编译进不同的配置头，CMake 在 configure 阶段强制校验该值（第 47–51 行），不允许非法组合。
- 应用可通过 `XCPLITE_CFG_OVERRIDE` 指向自定义 `.h` 文件，在默认配置之上二次覆盖 `OPTION_*` 宏；CMake 会拒绝与内置头同名的覆盖路径（第 97–101 行），防止意外替换 shipped 配置。
- 对外暴露命名空间别名 `xcplite::xcplite`（第 254 行），使消费者无论通过 `add_subdirectory`、`FetchContent` 还是 `find_package` 都能以相同方式 `target_link_libraries(... PRIVATE xcplite::xcplite)`。

### 3.2 第三方 C/C++ 依赖：FetchContent 拉源码
- 仅 FreeRTOS-Kernel 作为第三方 C 代码通过 `FetchContent_Declare` 在 `examples/freertos_demo/freertos_emu_demo/CMakeLists.txt` 中下载并在 configure 时获取（需网络访问）。
- 其他示例（如 SilKit demo）要求先独立安装 xcplite，再通过 `find_package` 引用，不在仓库内自动拉取。

### 3.3 Rust 工具依赖：Cargo + 锁定文件
- `tools/xcpclient/Cargo.toml` 声明了日志、CLI、序列化、ELF/DWARF 解析等 crate，并通过 `git = "https://github.com/vectorgrp/xcp-lite"` 引入上游 `xcp_registry` crate（注释中还保留了 fork 分支和本地 path 两种备选来源）。
- `Cargo.lock` 随仓库提交，锁定全部 crate 版本与 checksum，保证跨机器可重现构建。
- `tools/bintool` 是更轻量的独立 cargo crate，同样有 `Cargo.lock`。
- 顶层 CMake 通过 `XCPLITE_BUILD_RUST_TOOLS` 选项调用 `cargo build`（556–586 行），仅在检测到 `cargo` 可执行时才启用，且 Release/RelWithDebInfo 下自动加 `--release`。

### 3.4 安装与分发
- 安装产物包括：静态库、公共头（`inc/`）、`src/xcplib_cfg.h`、`platform.h`、`sockets.h`、`socket_raw_hal.h`、选中的配置覆盖头、以及 `xcpliteTargets.cmake` / `xcpliteConfig.cmake` / `xcpliteConfigVersion.cmake`（598–649 行）。
- 当作为子工程被消费时，`XCPLITE_INSTALL` 默认关闭（第 154 行），避免污染宿主项目的安装前缀。

## 4. 约定与约束

- **配置互斥**：`XCPLITE_CONFIGURATION` 必须为 `default | no_a2l | ptp | shm | rtos | raw` 之一，否则 CMake 直接 `message(FATAL_ERROR ...)` 终止配置（第 47–51 行）。
- **覆盖头命名保护**：`XCPLITE_CFG_OVERRIDE` 指定的文件名不得匹配 `^xcplib_(no_a2l_|ptp_|shm_|rtos_|raw_)?cfg\.h$`，防止误覆盖 shipped 配置（第 97–101 行）。
- **覆盖头仅用于 default 配置**：若指定了 `XCPLITE_CFG_OVERRIDE` 但 `XCPLITE_CONFIGURATION` 不是 `default`，CMake 报错（第 85–90 行）。
- **Rust 工具按需构建**：只有开启 `XCPLITE_BUILD_RUST_TOOLS=ON` 且 PATH 中存在 `cargo` 才会尝试构建 `xcpclient` 与 `bintool`，否则会发出 warning 并跳过（第 557–560 行）。
- **BPF demo 条件构建**：`bpf_demo` 需要 Linux 且找到 `libbpf`，找不到时仅打印 status 并跳过，不会导致配置失败（第 348–359 行）。
- **FreeRTOS 示例需网络**：`freertos_emu_demo` 在 configure 阶段通过 FetchContent 下载 FreeRTOS-Kernel，明确要求网络可达（第 412–416 行）。
- **Rust 依赖锁定**：`Cargo.lock` 随仓库提交，表明应通过锁文件保证可重现构建，而非每次重新解析版本。
- **上游 Rust crate 来源可切换**：`xcpclient` 对 `xcp_registry` 同时保留 git 远端、fork 分支、本地 path 三种来源的注释写法，便于开发者在离线或 fork 场景下切换（Cargo.toml 42–46 行）。

总体而言，该项目将 C/C++ 核心库的依赖最小化到系统库，通过 CMake 配置头实现功能裁剪；对少量第三方 C 代码使用 FetchContent 拉源码；对 Rust 工具使用 Cargo 并锁定版本。消费方既可以通过 `FetchContent` 直接从源码集成，也可以通过 `find_package` 引用已安装的版本，形成统一的依赖接入面。