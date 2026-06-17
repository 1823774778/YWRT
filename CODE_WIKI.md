# ImmortalWrt AI Edition — Code Wiki

> 本文档为 ImmortalWrt AI Edition（OpenWrt 衍生分支，高通平台专用）构建系统的结构化代码文档，覆盖项目整体架构、主要模块职责、关键类与函数说明、依赖关系以及项目运行方式。

---

## 目录

- [1. 项目概览](#1-项目概览)
- [2. 整体架构](#2-整体架构)
- [3. 顶层构建编排](#3-顶层构建编排)
- [4. 核心构建规则库（rules.mk 与 include/）](#4-核心构建规则库rulesmk-与-include)
- [5. 工具链模块（toolchain/）](#5-工具链模块toolchain)
- [6. 主机工具模块（tools/）](#6-主机工具模块tools)
- [7. 目标平台模块（target/）](#7-目标平台模块target)
- [8. 软件包模块（package/）](#8-软件包模块package)
- [9. 脚本模块（scripts/）](#9-脚本模块scripts)
- [10. 配置模块（config/）](#10-配置模块config)
- [11. 依赖关系总览](#11-依赖关系总览)
- [12. 项目运行方式](#12-项目运行方式)
- [13. 关键设计模式与约定](#13-关键设计模式与约定)

---

## 1. 项目概览

### 1.1 项目定位

ImmortalWrt AI Edition 是基于 OpenWrt/ImmortalWrt 的衍生发行版构建系统，主要特点：

- **高通平台专用**：`main` 分支仅支持高通平台，带满血 NSS（Network Subsystem Services）驱动
- **多平台通用**：`owrt` 分支可编译高通平台但无 NSS 驱动
- **AI 自动适配**：高通 6.18 内核由 GPT5.5 全程自动适配
- **双包管理器**：同时支持传统 OPKG（`.ipk`）与现代 APK（`.apk`）
- **可重现构建**：通过 `SOURCE_DATE_EPOCH`、文件排序、版本固定等机制保证构建可复现

### 1.2 顶层目录结构

```
/workspace/
├── Makefile              # 顶层构建入口（双阶段加载）
├── Config.in             # Kconfig 主配置入口
├── rules.mk              # 核心构建变量与宏库
├── feeds.conf.default    # 默认 feeds 配置
├── config/               # Kconfig 配置定义文件
├── include/              # Makefile 公共能力库（.mk 文件）
├── toolchain/            # 交叉工具链构建（binutils/gcc/glibc/musl）
├── tools/                # 主机端工具构建（约 60 个工具）
├── target/               # 目标平台支持（Linux 内核 + 镜像 + SDK + IB）
├── package/              # 软件包源码（base-files + 各类包）
├── scripts/              # 辅助脚本（下载、元数据、打包、配置工具）
└── LICENSES/             # 开源许可证文本
```

### 1.3 许可证

GPL-2.0-only（详见 `COPYING`），`LICENSES/` 目录包含项目使用的所有许可证文本（BSD-2-Clause、BSD-3-Clause、GPL-1.0、GPL-2.0、ISC、MIT、Linux-syscall-note 等）。

---

## 2. 整体架构

### 2.1 双阶段加载设计

构建系统采用 GNU Make 的"双阶段加载"模式：

```
┌─────────────────────────────────────────────────────────────┐
│ 阶段 1：配置阶段（OPENWRT_BUILD 未设置）                      │
│  - 加载 include/debug.mk、include/depends.mk、include/toplevel.mk │
│  - 处理 Kconfig 交互（menuconfig/defconfig/oldconfig）       │
│  - 生成元数据（.packageinfo/.targetinfo）                    │
│  - 主机前置检查（prereq-build.mk）                           │
│  - 通过 %:: 通配规则重新进入 Makefile（设置 OPENWRT_BUILD=1） │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ 阶段 2：构建阶段（OPENWRT_BUILD=1）                          │
│  - 加载 rules.mk（全局变量/宏）                              │
│  - 加载 include/depends.mk、include/subdir.mk                │
│  - 加载四大子系统：target/、package/、tools/、toolchain/      │
│  - 通过 subdir.mk 递归遍历子目录                             │
│  - 通过 stampfile 宏管理构建状态                             │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 构建依赖链

顶层 [Makefile](file:///workspace/Makefile) 第 46-51 行定义了核心构建顺序：

```
tools/stamp-compile
       │
       ▼
toolchain/stamp-compile
       │
       ▼
target/stamp-compile  ← 依赖 tools + toolchain
       │
       ▼
package/stamp-compile ← 依赖 target
       │
       ▼
package/stamp-install
       │
       ▼
target/stamp-install  ← 依赖 package 编译与安装
```

### 2.3 架构关系图

```
用户 make <target>
        │
        ▼
   /workspace/Makefile (OPENWRT_BUILD != 1)
        │ include
        ▼
   include/toplevel.mk
        │ ├─ prepare-tmpinfo → scan.mk + metadata.pl → .packageinfo/.targetinfo/Kconfig .in
        │ ├─ scripts/config/conf (Kconfig) → .config
        │ ├─ prereq-build.mk → include/prereq.mk (主机工具检查)
        │ └─ %:: 通配规则 → SUBMAKE -r $@ (重新进入 Makefile, OPENWRT_BUILD=1)
        │
        ▼
   /workspace/Makefile (OPENWRT_BUILD=1)
        │ include
        ├─ rules.mk          (全局变量/宏/路径/工具链)
        ├─ depends.mk        (依赖跟踪)
        ├─ subdir.mk         (递归遍历 + stampfile)
        ├─ target/Makefile   ─┐
        ├─ package/Makefile   │ 通过 subdir.mk 遍历各子目录
        ├─ tools/Makefile     │
        └─ toolchain/Makefile ─┘
              │
              ▼
   各子目录 Makefile 通过宏接入框架:
   ├─ tools/*/Makefile     → HostBuild (host-build.mk + download.mk)
   ├─ toolchain/*/Makefile → HostBuild + toolchain-build.mk
   ├─ package/*/Makefile   → BuildPackage (package.mk + download.mk + kernel.mk for kmod)
   └─ target/linux/*/Makefile → BuildTarget/BuildKernel (target.mk + kernel.mk + kernel-build.mk)
                                  └─ image/Makefile → BuildImage (image.mk + rootfs.mk + feeds.mk)
```

---

## 3. 顶层构建编排

### 3.1 [Makefile](file:///workspace/Makefile) — 顶层构建入口

**职责**：整个构建系统的入口与总调度器，定义构建阶段之间的依赖顺序，暴露用户可调用的顶层目标。

**关键内容**：

- 第 5 行：`TOPDIR:=${CURDIR}`，作为整个构建的根路径基准
- 第 13 行：强制要求路径中不得包含空格（OpenWrt Makefile 的硬性约束）
- 第 22-33 行（配置阶段）：首次进入时设置 `_SINGLE`、覆盖 `OPENWRT_BUILD=1` 并导出，加载 `debug.mk`、`depends.mk`、`toplevel.mk`
- 第 34-44 行（构建阶段）：加载 `rules.mk`、`depends.mk`、`subdir.mk`，并 include 四大子系统 Makefile

**关键目标**：

| 目标 | 说明 |
|------|------|
| `world` | 完整构建：prepare → target 编译 → package 编译/安装 → target 安装 → package/index → json_overview_image_info → checksum |
| `clean` | 清理 `BUILD_DIR`、`STAGING_DIR`、`BIN_DIR` 等 |
| `targetclean` | 在 clean 基础上清理 `TOOLCHAIN_DIR` |
| `dirclean` | 彻底清理，包括 `STAGING_DIR_HOST`、`TMP_DIR`、scripts/config |
| `cacheclean` | 清理 ccache 缓存 |
| `prereq` | 检查前置条件，包括架构 site 配置文件存在性 |
| `prepare` | 准备阶段：`.config` + tools 编译 + toolchain 编译 + buildinfo |
| `buildinfo` | 生成构建元信息（diffconfig + buildversion + feedsversion） |
| `checksum` | 生成 `BIN_DIR` 下文件的 sha256 校验和 |

### 3.2 [Config.in](file:///workspace/Config.in) — Kconfig 主入口

**职责**：定义 Kconfig 主菜单 "ImmortalWRT Configuration"，source 各子配置文件。

**source 链**：

- `target/Config.in` — 目标平台与硬件特性
- `config/Config-images.in` — 镜像配置
- `config/Config-build.in` — 全局构建设置
- `config/Config-devel.in` — 开发者选项
- `toolchain/Config.in` — 工具链配置
- `target/imagebuilder/Config.in` — Image Builder
- `target/sdk/Config.in` — SDK
- `target/toolchain/Config.in` — 预编译工具链
- `tmp/.config-package.in` — 动态生成的包配置
- `config/Config-ipq.in` — 高通 IPQ 专属选项

### 3.3 [include/toplevel.mk](file:///workspace/include/toplevel.mk) — 顶层构建逻辑

**职责**：处理配置系统交互、临时信息生成、前置检查触发、各种 `*config` 目标、内核配置目标、下载、清理，以及最关键的"通配目标" `%::`。

**关键机制**：

- `PREP_MK= OPENWRT_BUILD= QUIET=0`：在配置阶段临时清除 `OPENWRT_BUILD`
- 通过 `scripts/getver.sh` 和 `get_source_date_epoch.sh` 获取版本号和源码时间戳
- unexport 一系列可能干扰构建的环境变量（`P4PORT`、`QUILT_PATCHES`、`CROSS_COMPILE`、`ARCH`、`CFLAGS`、`LDFLAGS` 等）
- `prepare-tmpinfo`：核心元数据生成步骤——调用 `include/scan.mk` 扫描 `package/` 和 `target/linux/` 目录生成 `.packageinfo` 和 `.targetinfo`，再通过 `scripts/package-metadata.pl` 和 `scripts/target-metadata.pl` 生成 Kconfig `.in` 文件、`.packagedeps`、`.packageauxvars`、`.packageusergroup`
- `%::` 通配规则（第 230 行）：任何未明确匹配的目标都会先执行 `prereq`、用 `config/conf --defconfig` 校验 `.config` 同步性，再通过 `SUBMAKE -r $@` 重新进入 Makefile

**关键目标**：

| 目标 | 说明 |
|------|------|
| `prepare-tmpinfo` | 扫描包/目标 Makefile 生成元数据 |
| `.config` | 若不存在则引导用户进入 menuconfig |
| `config`/`defconfig`/`oldconfig`/`menuconfig`/`nconfig`/`xconfig` | Kconfig 配置界面 |
| `kernel_oldconfig`/`kernel_menuconfig`/`kernel_nconfig`/`kernel_xconfig` | 内核配置 |
| `download` | 依次下载 tools/toolchain/package/target 的源码 |
| `package/symlinks*` | feeds 管理（update/install/uninstall） |
| `distclean` | 彻底清理 |

---

## 4. 核心构建规则库（rules.mk 与 include/）

### 4.1 [rules.mk](file:///workspace/rules.mk) — 核心变量与宏库

**职责**：定义整个构建系统共享的基础变量、路径、工具链引用、字符串处理宏和编译选项。通过 `__rules_inc` 守卫保证只加载一次。

**关键宏/函数**：

| 宏 | 说明 |
|----|------|
| `qstrip` | 去除字符串中的引号和 `#`，用于清理 Kconfig 变量值 |
| `merge` | 合并字符串（去除空格） |
| `confvar` | 计算变量列表的 MD5 哈希（用于触发重建） |
| `strip_last` | 去除文件名最后一个扩展名 |
| `toupper`/`tolower` | 字符大小写转换 |
| `version_abbrev` | 版本号截断为 8 字符 |
| `DefaultTargets` | 从 `DEFAULT_SUBDIR_TARGETS` 生成默认目标 |
| `shvar`/`shexport` | 将 shell 函数结果导出为变量 |
| `locked` | 通过 `flock` 实现并发互斥（多线程构建保护） |
| `sha256sums` | 生成 sha256 校验和 |
| `commitcount` | AUTORELEASE/COMMITCOUNT 实现 |
| `abi_version_str` | ABI 版本字符串归一化 |

**关键变量分组**：

- **架构相关**：`ARCH`、`ARCH_PACKAGES`、`BOARD`、`SUBTARGET`、`TARGET_OPTIMIZATION`、`TARGET_SUFFIX`、`ARCH_SUFFIX`、`GCC_ARCH`、`OPTIMIZE_FOR_CPU`、`FPIC`
- **目录布局**：`DL_DIR`、`OUTPUT_DIR`、`BIN_DIR`、`INCLUDE_DIR`、`SCRIPT_DIR`、`BUILD_DIR_BASE`、`BUILD_DIR`、`BUILD_DIR_TOOLCHAIN`、`BUILD_DIR_HOST`、`STAGING_DIR`、`STAGING_DIR_HOST`、`STAGING_DIR_HOSTPKG`、`STAGING_DIR_ROOT`、`STAGING_DIR_IMAGE`、`TOOLCHAIN_DIR`、`STAMP_DIR`、`TARGET_DIR`、`TARGET_ROOTFS_DIR`、`PKG_INFO_DIR`、`BUILD_LOG_DIR`、`TMP_DIR`
- **工具链命名**：`GNU_TARGET_NAME`、`REAL_GNU_TARGET_NAME`、`TARGET_CROSS`、`TARGET_DIR_NAME`、`TOOLCHAIN_DIR_NAME`、`DIR_SUFFIX`
- **编译器/工具**：`HOSTCC`、`HOSTCXX`、`TARGET_CC`、`TARGET_CXX`、`TARGET_AR`、`TARGET_LD`、`MKHASH`、`SED`、`ESED`、`FAKEROOT`、`PKG_CONFIG`、`NINJA`
- **编译选项**：`TARGET_CFLAGS`、`TARGET_CXXFLAGS`、`TARGET_LDFLAGS`、`TARGET_PATH`、`TARGET_CONFIGURE_OPTS`、`HOST_CFLAGS`、`HOST_LDFLAGS`
- **安装命令**：`INSTALL_BIN`、`INSTALL_DIR`、`INSTALL_DATA`、`INSTALL_CONF`、`INSTALL_SUID`
- **Strip 相关**：`STRIP`、`RSTRIP`（调用 `scripts/rstrip.sh`）
- **签名密钥**：`BUILD_KEY`、`BUILD_KEY_APK_SEC`、`BUILD_KEY_APK_PUB`

### 4.2 [include/subdir.mk](file:///workspace/include/subdir.mk) — 子目录递归遍历

**职责**：实现构建系统的递归子目录遍历机制，是 tools/toolchain/package/target 四大子系统能够按目录树构建的核心。

**关键宏**：

- `subdir`（核心宏）：遍历 `$(1)/builddirs` 中每个子目录 `bd`，对每个 target 和 buildtype 生成规则，调用 `log_make` 执行子目录 make，支持变体（variants）和目录别名
- `stampfile`：为子目录创建 stamp 文件机制——`$(1)/stamp-$(3)` 依赖 `TMP_DIR/.build` 和指定依赖，通过 `timestamp.pl` 比较时间戳决定是否重建
- `subtarget`：为某 target 生成聚合规则
- `log_make`：带计时（`time.pl`）和日志（`BUILD_LOG` 时 tee 到文件）的子 make 调用
- `rebuild_check`：`CONFIG_AUTOREMOVE` 时先用 `check-depends` 检查，再用 `make -q` 判断是否需要重建

**builddirs 体系**：每个子系统 Makefile 设置 `$(name)/builddirs`、`$(name)/builddirs-default`、`$(name)/builddirs-$(target)` 来声明子目录。

### 4.3 [include/package.mk](file:///workspace/include/package.mk) — 软件包构建框架

**职责**：定义软件包（ipkg/apk）的构建生命周期：下载 → 准备 → 配置 → 编译 → 安装到 staging → 打包。

**关键宏**：

- `BuildPackage`（核心入口）：执行 `Package/Default` 与 `Package/$(1)` 的 eval，校验 TITLE/CATEGORY/SECTION/VERSION 字段，加入 `BUILD_PACKAGES`，根据 `PKG_TARGETS`（默认 ipkg）调用 `BuildTarget/$(target)`，最后调用 `Build/DefaultTargets` 注册下载与核心目标
- `Build/CoreTargets`：定义四大 stamp 目标的完整规则
- `pkg_build_flag`：解析 `PKG_BUILD_FLAGS`（no-iremap/no-mips16/gc-sections/lto/no-lto/no-mold 等）
- `find_library_dependencies`：递归 5 层（dep0..dep4）查找包的库依赖
- `CleanStaging`：通过 `scripts/clean-package.sh` 按 `.list` 文件清理 staging 中本包安装的文件

**构建生命周期（CoreTargets）**：

1. `STAMP_PREPARED`：清空 build_dir → Pre hooks → `Build/Prepare` → Post hooks
2. `STAMP_CONFIGURED`：清旧 configured stamp → CleanStaging → Pre → `Build/Configure` → Post
3. `STAMP_BUILT`：Pre → `Build/Compile` → `Build/Install` → Post
4. `STAMP_INSTALLED`：在 `TMP_DIR/stage-$(PKG_DIR_NAME)` 临时目录中执行 `Build/InstallDev`，记录文件清单，加锁移动到 `STAGING_DIR`

**include 的子框架**：`download.mk`、`hardening.mk`、`prereq.mk`、`unpack.mk`、`depends.mk`、`quilt.mk`、`package-defaults.mk`、`package-dumpinfo.mk`、`package-pack.mk`、`package-bin.mk`、`autotools.mk`

### 4.4 [include/host-build.mk](file:///workspace/include/host-build.mk) — 主机端构建

**职责**：定义在主机（build host）上运行的工具的构建框架，是 `tools/` 和 `toolchain/` 中所有 `HostBuild` 包的基类。

**关键宏**：

- `HostBuild`（入口宏）：调用 `HostBuild/Core` 并注册下载
- `HostBuild/Core`：定义四大 stamp 规则（Prepared→Configured→Built→Installed），含 hooks 机制
- `Host/Prepare/Default`：解包 + 复制 `./src/` + `Host/Patch`
- `Host/Configure/Default`：复制 `config.guess`/`config.sub`，运行 configure
- `Host/Compile/Default`：`make -C $(HOST_BUILD_DIR)/$(HOST_MAKE_PATH)`
- `Host/Install/Default`：`make install`

**关键机制**：

- `HOST_BUILD_PREFIX` 的二分：package 构建时用 `STAGING_DIR_HOSTPKG`（隔离），否则用 `STAGING_DIR_HOST`
- `HOST_STAMP_PROGRAMS`：支持一个 Makefile 产出多个程序（`PKG_PROGRAMS`）
- `FORCE_HOST_INSTALL`：强制重装

### 4.5 [include/target.mk](file:///workspace/include/target.mk) — 目标平台定义框架

**职责**：定义硬件目标（BOARD/SUBTARGET）的抽象层，管理默认包集合、CPU 类型/CFLAGS、内核配置文件选择、Profile（设备配置文件）机制，以及 `BuildTarget` 宏。

**关键内容**：

- `DEVICE_TYPE`：默认 router，可为 basic/nas/router
- `DEFAULT_PACKAGES` 及其变体：按设备类型分组的默认包集合。ImmortalWrt 特有的 `DEFAULT_PACKAGES.tweak`（含 luci、autocore、default-settings-chn 等中国本土化包）
- `target_conf`：把 BOARD/SUBTARGET 字符串归一化为合法 Kconfig 名
- `PLATFORM_DIR`/`PLATFORM_SUBDIR`：定位 `target/linux/$(BOARD)` 目录，支持 feeds 覆盖
- `Profile`/`ProfileDefault`：设备 Profile 机制，定义 NAME/PRIORITY/PACKAGES/Description
- 内核配置文件管理：`GENERIC_PLATFORM_DIR`、`GENERIC_BACKUP_DIR`、`GENERIC_PATCH_DIR`、`GENERIC_HACK_DIR`、`GENERIC_FILES_DIR`；`LINUX_KCONFIG_LIST`/`LINUX_RECONFIG_LIST`/`LINUX_RECONFIG_TARGET` 通过 `kconfig.pl` 合并 generic+target+subtarget 配置
- `BuildTargets/DumpCurrent`：DUMP 模式下输出目标元信息供 `scan.mk` 收集
- `TARGET_BUILD=1` 时 include `kernel-build.mk`，把 `BuildTarget` 设为 `BuildKernel`

### 4.6 [include/image.mk](file:///workspace/include/image.mk) — 镜像生成框架

**职责**：把内核、rootfs、设备树打包成最终可刷写的固件镜像。是构建系统最复杂的文件之一（1025 行）。

**关键宏**：

- `Device`（设备定义入口）：依次调用 InitProfile/Init/Default/$(1)/Check/Build
- `BuildImage`（镜像构建总入口）：定义 `download/prepare/compile/kernel_prepare/install-images/install` 目标，遍历 `TARGET_DEVICES` 调用 `Device`
- `Device/Build/kernel`/`image`/`artifact`/`initramfs`/`compile`/`dtb`/`dtbo`：定义各产物的构建规则
- `split_args`/`build_cmd`/`concat_cmd`：把 `|` 分隔的命令链解析为对 `Build/xxx` 宏的连续调用
- `Image/mkfs/*`：各文件系统生成——jffs2（含 nand 变体）、squashfs（含 SELinux 标签支持）、ubifs、ext4、erofs、targz
- `Image/BuildDTB`/`Image/BuildDTBO`：通过 `cpp` 预处理 + `dtc` 编译设备树

**关键机制**：

- per-device rootfs（`TARGET_PER_DEVICE_ROOTFS`）：每个设备可有不同包集合，通过 `mkfs_packages_id` 复用相同包集合的 rootfs
- IB（ImageBuilder）模式：跳过编译，直接从预构建产物组装
- APK 与 OPKG 双支持：根据 `CONFIG_USE_APK` 选择包管理器
- JSON 元数据：每个镜像生成 `.json` 供 ImageBuilder 和 CI 使用
- CycloneDX SBOM：可选生成 `.bom.cdx.json`

### 4.7 [include/kernel.mk](file:///workspace/include/kernel.mk) — 内核构建

**职责**：定义 Linux 内核构建所需的所有变量、kmod 包框架（`KernelPackage` 宏）、内核模块符号收集、内核版本比较工具。

**关键宏**：

- `KernelPackage`（核心宏）：定义 kmod 包。依次 eval `KernelPackage/Defaults`、`KernelPackage/$(1)`、`KernelPackage/$(1)/$(BOARD)`、`KernelPackage/$(1)/$(BOARD)/$(SUBTARGET)`，生成 `Package/kmod-$(1)` 定义（含 `EXTRA_DEPENDS` 锁定内核版本+vermagic），处理 KCONFIG 过滤、模块安装、AUTOLOAD，最后调用 `BuildPackage`
- `collect_module_symvers`：收集外部模块的 `Module.symvers` 到 `PKG_SYMVERS_DIR`，供后续模块构建使用
- `ModuleAutoLoad`：生成 `/etc/modules.d/` 和 `/etc/modules-boot.d/` 下的模块加载配置
- `AutoLoad`/`AutoProbe`：模块自动加载声明
- `CompareKernelPatchVer`/`kernel_patchver_gt/ge/eq/le/lt`：内核版本比较

**关键变量**：`LINUX_DIR`、`LINUX_VERSION`、`LINUX_VERMAGIC`、`LINUX_KARCH`、`KERNEL_BUILD_DIR`、`KERNEL_MAKE_FLAGS`、`KERNEL_MAKEOPTS`、`MODULES_SUBDIR`、`TARGET_MODULES_DIR`、`PKG_SYMVERS_DIR`

**LINUX_KARCH 映射**：aarch64→arm64、armeb→arm、loongarch64→loongarch、mipsel/mips64→mips、powerpc64→powerpc、riscv64→riscv、i386/x86_64→x86、uml→um

### 4.8 [include/download.mk](file:///workspace/include/download.mk) — 下载机制

**职责**：定义源码下载框架，支持多种协议（http/https/ftp/git/svn/hg/bzr/darcs/file）、镜像回退、哈希校验、`make check` 时的哈希审计与自动修复。

**关键宏**：

- `Download`（入口宏）：eval Defaults + `Download/$(1)`，校验必填字段，建立 `$(DL_DIR)/$(FILE)` 目标，通过 `locked` 加锁调用对应 `DownloadMethod`
- `dl_method`：根据 URL 前缀（`@OPENWRT`/`@GITHUB`/`@GNU`/`@KERNEL`/`@SF` 等）和 PROTO 推断下载方法
- `DownloadMethod/*`：各协议的具体下载实现——`default`（调用 `download.pl`）、`git`/`rawgit`（clone+checkout+git archive+submodule）、`github_archive`（调用 `dl_github_archive.py`）、`svn`/`bzr`/`hg`/`darcs`

**关键机制**：

- 镜像优先（`wrap_mirror`）：`MIRROR_HASH` 存在时先尝试镜像，失败再走原始方法
- 并发安全：通过 `locked` 宏（flock）保证同一文件并发下载互斥
- `make check`：审计所有包的哈希正确性，`FIXUP=1` 可自动修复

### 4.9 [include/feeds.mk](file:///workspace/include/feeds.mk) — Feeds 管理

**职责**：管理外部软件源（feeds）的包索引、仓库源列表生成、ABI 后缀解析。

**关键宏**：

- `FeedPackageDir`：根据 feed 配置返回包应存放的目录
- `FeedSourcesAppendOPKG`/`FeedSourcesAppendAPK`：生成 `/etc/opkg/distfeeds.conf` 或 `/etc/apk/repositories.d/distfiles.list` 内容
- `GetABISuffix`/`FormatABISuffix`：从 `ABIV_$(1)` 或 `staging_dir/pkginfo/$(1).version` 读取 ABI 版本，按规则格式化为后缀

### 4.10 [include/quilt.mk](file:///workspace/include/quilt.mk) — Quilt 补丁管理

**职责**：集成 quilt 工具，支持对源码树增量打补丁、refresh 回 patch 目录，覆盖 package/host/kernel 三种场景。

**关键宏**：

- `Quilt/Template`（核心模板）：定义 `STAMP_CHECKED` 依赖、`quilt push -a`、series 排序校验、`refresh`、`update` 目标
- `PatchDir/Quilt`：把外部 patch 目录的 series 与补丁复制到构建树 `patches/` 下
- `PatchDir/Default`：非 quilt 模式直接用 `$(KPATCH)` 应用 series 或整个目录
- `Host/Patch/Default`、`Build/Patch/Default`、`Kernel/Patch/Default`：分别对应三种场景的补丁应用入口

### 4.11 [include/rootfs.mk](file:///workspace/include/rootfs.mk) — 根文件系统准备

**职责**：定义 rootfs 后处理流程，包括 mklibs 裁剪、opkg/apk 离线安装包装、postinst 脚本执行、init.d 启用/禁用、清理。

**关键宏**：

- `prepare_rootfs`（核心）：参数 (1:target dir, 2:可选覆盖目录, 3:禁用 init.d 列表)
  - 复制覆盖目录、建 `/etc/rc.d`、`/var/lock`
  - APK 模式：解压 `lib/apk/db/scripts.tar` 取出 `*.post-install`；OPKG 模式：用 `usr/lib/opkg/info/*.postinst`
  - 逐个执行 postinst，失败则退出
  - 遍历 `/etc/init.d/*`，对含 `#!/bin/sh /etc/rc.common` 的脚本，按第 3 参数决定 enable/disable
  - 清理 `.svn/.git/.#*`、`/boot`、`/tmp/*`、postinst 残留、opkg lists、lock 文件
  - 调用 `clean_ipkg` 与 `mklibs`
  - 用 `SOURCE_DATE_EPOCH` 统一 touch 所有文件时间戳（可复现构建）
- `mklibs`：`CONFIG_USE_MKLIBS` 时裁剪出最小库集合
- `clean_ipkg`：`CONFIG_CLEAN_IPKG` 时删除 opkg info 中非 control 文件

### 4.12 [include/depends.mk](file:///workspace/include/depends.mk) — 依赖追踪

**职责**：实现基于文件时间戳与 md5 的子树依赖检测，驱动 `AUTOREBUILD`。

**关键宏**：

- `rdep`（核心）：参数为 (1:目录/文件, 2:目标, 3:临时文件, 4:find 选项)
  - 声明 `.PRECIOUS: $(2)`、`.SILENT: $(2)_check`
  - 目标已存在时：计算 md5 写 `.1`，与旧 md5 diff；调用 `scripts/timestamp.pl` 比较；一致则 `touch -r` 保持时间戳并打印 "No need to rebuild"，否则触发重建
- `find_md5`：列出文件 + mtime，sort 后整体 md5
- `find_md5_reproducible`：可复现版本（按内容 md5 聚合，忽略 mtime）

### 4.13 [include/prereq.mk](file:///workspace/include/prereq.mk) 与 [include/prereq-build.mk](file:///workspace/include/prereq-build.mk) — 前置检查

**职责**：定义主机端前置依赖检查的通用框架。

**关键宏**：

- `Require`（核心）：为每个检查项生成 `prereq-$(1)` 目标，通过 `NO_TRACE_MAKE check-$(1)` 执行检查
- `RequireCommand`：检查命令是否存在
- `RequireHeader`/`RequireCHeader`：检查头文件存在性/编译测试
- `TestHostCommand`：执行任意 shell 测试命令
- `SetupHostCommand`（最常用）：在 `STAGING_DIR_HOST/bin` 下为命令创建符号链接，统一命令名

**prereq-build.mk 检查项**：

- 基础：`true`/`false`、GNU make ≥4.1、case-sensitive-fs、proper-umask
- 工具链：`gcc`/`g++` ≥10、`working-gcc`/`working-g++`、`ncurses.h`、`git` ≥1.7.12.2、`rsync`
- Perl 模块：Data::Dumper、FindBin、File::Copy、File::Compare、Thread::Queue、IPC::Cmd
- GNU 工具：tar、find、bash、xargs、patch、diff、cp、seq、awk、grep、getopt、realpath、stat、gzip、unzip、bzip2、wget、install、perl 5.x、python ≥3.8、file、which
- Linux 专属头文件（musl 兼容）：argp.h、fts.h、obstack.h、libintl.h
- 内置工具：`mkhash`（由 `scripts/mkhash.c` 编译）、`xxd`

### 4.14 [include/version.mk](file:///workspace/include/version.mk) 与 [include/kernel-version.mk](file:///workspace/include/kernel-version.mk) — 版本管理

**version.mk**：

- 集中定义发行版版本元数据：`VERSION_NUMBER`（默认 SNAPSHOT）、`VERSION_CODE`（默认 REVISION）、`VERSION_REPO`（默认 `https://downloads.immortalwrt.org/snapshots`）、`VERSION_DIST`（默认 ImmortalWRT）等
- `VERSION_SED_SCRIPT`：一段 sed 命令，把 `%U %V %v %C %c %D %d %R %T %S %A %t %M %m %b %u %s %f %P %h %B` 等占位符替换为对应版本字段
- `VERSION_TAINT_SPECS`：定义一组 taint 规则（如 `-ALL_KMODS:no-all`、`-IPV6:no-ipv6`、`+USE_GLIBC:glibc`）

**kernel-version.mk**：

- 根据 `KERNEL_PATCHVER` 与可选的 `KERNEL_TESTING_PATCHVER` 计算实际 `LINUX_VERSION`、`KERNEL`、`LINUX_KERNEL_HASH`
- 若 `CONFIG_TESTING_KERNEL` 定义，则 `KERNEL_PATCHVER` 切换为 testing 版本
- 从 `$(GENERIC_PLATFORM_DIR)/kernel-$(KERNEL_PATCHVER)` 文件读取内核细节（版本号、哈希、下载 URI）
- 支持 `CONFIG_KERNEL_GIT_CLONE_URI` 直接克隆 git 仓库

### 4.15 [include/site/](file:///workspace/include/site) — 架构专属 autoconf 配置

**职责**：为每个目标架构（及宿主 OS）提供预计算的 autoconf cache 变量（`ac_cv_*`），避免在交叉编译时运行那些无法正确探测目标特性的 configure 测试。

**目录内容**（21 个文件）：

- `linux`：Linux 通用基础（约 80 行，定义 `ac_cv_func_*`/`ac_cv_type_*`/`ac_cv_have_*` 等）
- `darwin`：macOS 兼容（覆盖 `futimens`/`utimensat` 为 no）
- 各架构文件均以 `. $TOPDIR/include/site/linux` 开头继承通用基础，再补充字节序与各类型 sizeof：
  - 小端 64 位：`aarch64`、`x86_64`、`riscv64`、`loongarch64`、`mips64el`、`powerpc64`
  - 小端 32 位：`arm`、`i386`、`i486`、`i686`、`mipsel`
  - 大端：`armeb`、`mips`、`mips64`、`powerpc`、`sparc`、`aarch64_be`

### 4.16 其他 include 文件

| 文件 | 职责 |
|------|------|
| [include/verbose.mk](file:///workspace/include/verbose.mk) | 输出静默控制、彩色高亮、`MESSAGE`/`ERROR_MESSAGE` 宏、`SUBMAKE` 包装 |
| [include/debug.mk](file:///workspace/include/debug.mk) | 按子目录过滤的调试输出能力（`DEBUG` 变量字符：d/t/l/r/v） |
| [include/default-packages.mk](file:///workspace/include/default-packages.mk) | 根据 `CONFIG_USE_APK` 决定默认包管理器（apk-openssl 或 opkg） |
| [include/hardening.mk](file:///workspace/include/hardening.mk) | 把加固 choice 翻译为 `TARGET_CFLAGS`/`TARGET_LDFLAGS`（PIE/SSP/FORTIFY/RELRO/RELR/FANALYZER） |
| [include/nls.mk](file:///workspace/include/nls.mk) | 根据 `CONFIG_BUILD_NLS` 切换完整版与 stub 版 iconv/intl |
| [include/netfilter.mk](file:///workspace/include/netfilter.mk) | Netfilter 模块映射表（内核 Kconfig 符号 ↔ `.ko`/用户态插件文件名） |
| [include/openssl-module.mk](file:///workspace/include/openssl-module.mk) | OpenSSL engine/provider 子包打包助手 |
| [include/scan.mk](file:///workspace/include/scan.mk) | 递归扫描 `package/` 或 `target/` 目录收集元数据 |
| [include/unpack.mk](file:///workspace/include/unpack.mk) | 根据源码包扩展名自动选择解压命令 |
| [include/toolchain-build.mk](file:///workspace/include/toolchain-build.mk) | 工具链构建的薄封装，禁用自动重建/清理 |
| [include/kernel-build.mk](file:///workspace/include/kernel-build.mk) | 内核构建实现 |
| [include/kernel-defaults.mk](file:///workspace/include/kernel-defaults.mk) | 内核默认配置 |
| [include/cmake.mk](file:///workspace/include/cmake.mk) | CMake 构建系统支持 |
| [include/meson.mk](file:///workspace/include/meson.mk) | Meson 构建系统支持 |
| [include/autotools.mk](file:///workspace/include/autotools.mk) | Autotools 构建系统支持 |
| [include/bpf.mk](file:///workspace/include/bpf.mk) | BPF 支持 |
| [include/optee-os.mk](file:///workspace/include/optee-os.mk) | OP-TEE 支持 |
| [include/trusted-firmware-a.mk](file:///workspace/include/trusted-firmware-a.mk) | TF-A 支持 |
| [include/u-boot.mk](file:///workspace/include/u-boot.mk) | U-Boot 支持 |
| [include/uclibc++.mk](file:///workspace/include/uclibc++.mk) | uClibc++ 支持 |
| [include/image-commands.mk](file:///workspace/include/image-commands.mk) | 镜像构建命令 |
| [include/package-defaults.mk](file:///workspace/include/package-defaults.mk) | 包构建默认值 |
| [include/package-dumpinfo.mk](file:///workspace/include/package-dumpinfo.mk) | 包信息输出 |
| [include/package-pack.mk](file:///workspace/include/package-pack.mk) | 包打包 |
| [include/package-bin.mk](file:///workspace/include/package-bin.mk) | 包二进制处理 |
| [include/package-seccomp.mk](file:///workspace/include/package-seccomp.mk) | seccomp 支持 |
| [include/openssl-module.mk](file:///workspace/include/openssl-module.mk) | OpenSSL 模块 |

---

## 5. 工具链模块（toolchain/）

### 5.1 [toolchain/Makefile](file:///workspace/toolchain/Makefile) — 工具链总调度

**职责**：协调整个交叉工具链的分阶段构建。

**七步构建流程**：

1. `binutils/compile` - 构建并安装 binutils
2. `gcc/minimal/compile` - 构建最小化 gcc（glibc 路径需要，用于步骤 3、4）
3. `kernel-headers/compile` - 安装内核头文件
4. `libc/headers/compile` - 构建并安装 libc 头文件
5. `gcc/initial/compile` - 构建初始 gcc
6. `libc/compile` - 构建最终 libc
7. `gcc/final/compile` - 构建最终 gcc

**musl 特殊路径**："For musl, steps 2 and 4 are skipped, and step 3 is done after 5"。这是因为 musl 不需要 bootstrap headers 阶段。

**依赖链**：

```
binutils/compile
       │
       ▼
gcc/initial/compile ← (glibc 路径先经 gcc/minimal)
       │
       ▼
kernel-headers/compile
       │
       ▼
$(LIBC)/compile  ← (glibc 路径先经 libc/headers)
       │
       ▼
gcc/final/compile
```

### 5.2 [toolchain/Config.in](file:///workspace/toolchain/Config.in) — 工具链配置

**关键配置组**：

- **TARGET_OPTIONS**：`TARGET_OPTIMIZATION`、`SOFT_FLOAT`、`USE_MIPS16`
- **BPF_TOOLCHAIN choice**：`BPF_TOOLCHAIN_NONE`/`PREBUILT`/`HOST`/`BUILD_LLVM`
- **EXTERNAL_TOOLCHAIN**：使用外部工具链（`NATIVE_TOOLCHAIN`、`TARGET_NAME`、`TOOLCHAIN_PREFIX`、`TOOLCHAIN_ROOT`、`TOOLCHAIN_LIBC_TYPE`、`EXTERNAL_GCC_VERSION`、`TOOLCHAIN_BIN_PATH`/`INC_PATH`/`LIB_PATH`）
- **TOOLCHAINOPTS**：`EXTRA_TARGET_ARCH`、`MIPS64_ABI`（n64/n32/o32）、`DWARVES`、`NASM`、`GDB`、`GDB_PYTHON`
- **C Library choice**：`LIBC_USE_GLIBC` 或 `LIBC_USE_MUSL`（默认 musl）
- **派生配置**：`LIBC`、`TARGET_SUFFIX`、`USE_GLIBC`、`USE_MUSL`、`SSP_SUPPORT`、`HAS_BPF_TOOLCHAIN` 等

### 5.3 [toolchain/binutils/](file:///workspace/toolchain/binutils/) — GNU binutils

**版本支持**：2.44（默认）、2.45.1、2.46.0

**关键配置**：

- `--target=$(REAL_GNU_TARGET_NAME)`、`--with-sysroot=$(TOOLCHAIN_DIR)`
- `--enable-lto`、`--enable-plugins`、`--disable-multilib`、`--disable-nls`
- `--with-system-zlib`、`--with-zstd`
- `GRAPHITE_CONFIGURE`：根据 `CONFIG_GCC_USE_GRAPHITE` 决定 `--with-isl` 或 `--without-isl --without-cloog`
- `SOFT_FLOAT_CONFIG_OPTION`：软浮点支持

### 5.4 [toolchain/gcc/](file:///workspace/toolchain/gcc/) — GCC 交叉编译器

**版本支持**：13.x（13.4.0）、14.x（14.3.0，默认）、15.x（15.2.0）

**三阶段构建**：

1. **minimal**：`--with-newlib --without-headers --enable-languages=c --disable-shared --disable-threads`，仅编译 `all-gcc all-target-libgcc`，为 glibc headers 阶段提供最小编译器
2. **initial**：`--with-newlib --with-sysroot=$(TOOLCHAIN_DIR) --enable-languages=c --disable-shared --disable-threads`，编译 `all-build-libiberty all-gcc all-target-libgcc`，为最终 libc 构建提供编译器
3. **final**：`--with-headers=$(TOOLCHAIN_DIR)/include --enable-languages=$(TARGET_LANGUAGES) --enable-shared --enable-threads --with-slibdir=$(TOOLCHAIN_DIR)/lib --enable-plugins --enable-lto --with-libelf`，完整功能的 GCC

**Config.in 选项**：`GCC_USE_GRAPHITE`、`EXTRA_GCC_CONFIG_OPTIONS`、`GCC_DEFAULT_PIE`、`GCC_DEFAULT_SSP`、`SJLJ_EXCEPTIONS`、`INSTALL_GFORTRAN`、`INSTALL_GCCGO`

### 5.5 [toolchain/glibc/](file:///workspace/toolchain/glibc/) — GNU C Library

**版本**：2.43（从 sourceware git 仓库获取）

**两阶段构建**：

1. **headers**：执行 `install-bootstrap-headers=yes install-headers`，复制 linux-dev 内容，编译 csu/subdir_lib，复制 crt1.o/crti.o/crtn.o，用 TARGET_CC 创建空 libc.so
2. **final**：`default-rpath="/lib:/usr/lib"` 全量构建，`install_root=$(TOOLCHAIN_DIR)`，修正 libc.so/libm.so/libpthread.so/libgcc_s.so 中的路径

**关键配置**：`--prefix= --host=$(REAL_GNU_TARGET_NAME) --with-headers=$(TOOLCHAIN_DIR)/include --disable-profile --disable-werror --without-gd --without-cvs --enable-add-ons --enable-kernel=6.6.0`

### 5.6 [toolchain/musl/](file:///workspace/toolchain/musl/) — musl libc（默认）

**版本**：1.2.6

**单阶段构建**：`--prefix=/ --host=$(GNU_HOST_NAME) --target=$(REAL_GNU_TARGET_NAME) --disable-gcc-wrapper --enable-debug --enable-optimize`

**Config.in**：`MUSL_DISABLE_CRYPT_SIZE_HACK`（包含 SHA256/SHA512/Blowfish 的 crypt() 支持，默认 y）

### 5.7 [toolchain/kernel-headers/](file:///workspace/toolchain/kernel-headers/) — 内核头文件

**职责**：安装 Linux 内核头文件供 libc 和用户空间使用。

- `PKG_NAME:=linux`，`PKG_VERSION:=$(LINUX_VERSION)`
- 支持 git 克隆源或标准 tarball
- `Host/Configure/all`：`headers_install` 到 `$(BUILD_DIR_TOOLCHAIN)/linux-dev/`
- `Host/Configure/lzma`（MIPS 特有）：复制 asm.h、regdef.h、asm-eva.h、isa-rev.h 用于 lzma-loader

### 5.8 [toolchain/gdb/](file:///workspace/toolchain/gdb/) — GDB 调试器

**版本**：16.3

**配置**：`--target=$(REAL_GNU_TARGET_NAME)`、`--enable-tui --disable-gdbtk --without-x`、`--enable-threads`、`--disable-binutils --disable-ld --disable-gas --disable-sim`、`CONFIG_GDB_PYTHON` 时 `--with-python`

### 5.9 [toolchain/wrapper/](file:///workspace/toolchain/wrapper/) — 外部工具链包装器

**职责**：为外部工具链创建包装器并测试其能力。

- `toolchain_util`：调用 `$(SCRIPT_DIR)/ext-toolchain.sh`
- `toolchain_test`：测试外部工具链是否支持某特性（softfloat/ipv6/wchar/threads）
- `Host/Prepare`：测试 SOFT_FLOAT、IPV6、NLS、libpthread 支持
- `Host/Install`：`--wrap "$(TOOLCHAIN_DIR)/bin"` 包装工具链二进制

### 5.10 其他工具链组件

| 组件 | 说明 |
|------|------|
| [toolchain/fortify-headers/](file:///workspace/toolchain/fortify-headers/) | 版本 1.1，提供 `_FORTIFY_SOURCE` 头文件 |
| [toolchain/mold/](file:///workspace/toolchain/mold/) | 现代链接器，由 `CONFIG_USE_MOLD` 触发 |
| [toolchain/nasm/](file:///workspace/toolchain/nasm/) | NASM 汇编器，仅 i386/x86_64 |
| [toolchain/info.mk](file:///workspace/toolchain/info.mk) | 模板文件，记录 TARGET_CROSS/GCC_VERSION/LIBC_TYPE 等 |
| [toolchain/build_version](file:///workspace/toolchain/build_version) | 内容为 `1` |

---

## 6. 主机工具模块（tools/）

### 6.1 [tools/Makefile](file:///workspace/tools/Makefile) — 工具构建协调器

**职责**：协调所有主机工具（host tools）的构建，这些工具运行在构建主机上用于辅助交叉编译。

**工具分类**：

- **始终构建**（`tools-y`）：autoconf、automake、cmake、fakeroot、flex、bison、libtool、lzma、meson、mkimage、mtd-utils、ninja、patchelf、quilt、squashfs4、xz、zlib 等约 40 个
- **条件构建**：
  - 工具链相关：`gmp`/`mpc`/`mpfr`（构建工具链时）、`isl`（Graphite 优化）
  - 压缩格式：`bzip2`/`lz4`/`lzop`（对应 initramfs 压缩）
  - 平台特定：`7z`（realtek）、`elftosb`/`sdimage`（mxs）、`cbootimage`（tegra）、`genext2fs`（apm821xx/gemini）、`lzma-old`/`squashfs3-lzma`（ath79）
  - 链接器：`mold`（`CONFIG_USE_MOLD`）、`llvm-bpf`（`CONFIG_USE_LLVM_BUILD`）、`sparse`（`CONFIG_USE_SPARSE`）

**依赖管理**（核心特性）：

- 显式声明工具间编译依赖，如 `$(curdir)/cmake/compile += $(curdir)/libressl/compile $(curdir)/ninja/compile ...`
- `tools-core`（libdeflate、patch、tar、zstd）作为所有工具的基础依赖
- `sed` 和 `flock` 作为核心工具的依赖
- 有 patches 目录的工具自动依赖 `patch/compile`
- ccache 启用时，除少数基础工具外所有工具依赖 `ccache/compile`

### 6.2 工具 Makefile 通用模式

所有工具 Makefile 遵循高度一致的模板：

```makefile
include $(TOPDIR)/rules.mk          # 引入构建规则

PKG_NAME:=...                       # 包名
PKG_VERSION:=...                    # 版本
PKG_SOURCE:=...                     # 源码包名
PKG_SOURCE_URL:=...                 # 下载地址
PKG_HASH:=...                       # SHA256 校验

include $(INCLUDE_DIR)/host-build.mk  # 引入主机构建框架

# 可选：配置变量和参数
HOST_CONFIGURE_VARS += ...
HOST_CONFIGURE_ARGS += ...

# 可选：覆盖默认的 Host/Configure, Host/Compile, Host/Install, Host/Clean
define Host/Configure
  ...
endef

$(eval $(call HostBuild))           # 求值 HostBuild 宏
```

### 6.3 代表性工具

| 工具 | 版本 | 说明 |
|------|------|------|
| [tools/ccache/](file:///workspace/tools/ccache/) | 4.13.3 | 使用 cmake.mk 构建，禁用文档/测试/Redis 后端 |
| [tools/cmake/](file:///workspace/tools/cmake/) | 4.3.3 | 自身用 bootstrap + Ninja 构建，使用系统库 |
| [tools/fakeroot/](file:///workspace/tools/fakeroot/) | 1.38.1 | 使用 `PKG_FIXUP:=autoreconf`，禁用 libcap，使用 TCP IPC |
| [tools/mkimage/](file:///workspace/tools/mkimage/) | - | 来源是 U-Boot 源码包，运行 `tools-only_defconfig` |
| [tools/patch/](file:///workspace/tools/patch/) | 2.8 | GNU patch，最简单的工具之一 |
| [tools/zlib/](file:///workspace/tools/zlib/) | 1.3.2 | 静态构建，添加 `HOST_FPIC` |
| [tools/quilt/](file:///workspace/tools/quilt/) | - | 补丁管理工具，被 quilt.mk 使用 |
| [tools/mkhash/](file:///workspace/tools/mkhash/) | - | 哈希计算工具（C 源码 `scripts/mkhash.c`） |
| [tools/llvm-bpf/](file:///workspace/tools/llvm-bpf/) | - | LLVM BPF 工具链 |
| [tools/squashfs4/](file:///workspace/tools/squashfs4/) | - | SquashFS 工具 |
| [tools/lzma/](file:///workspace/tools/lzma/) | - | LZMA 压缩工具 |
| [tools/xz/](file:///workspace/tools/xz/) | - | XZ 压缩工具 |
| [tools/zstd/](file:///workspace/tools/zstd/) | - | Zstandard 压缩工具 |

**完整工具列表**（约 60 个）：7z、autoconf、autoconf-archive、automake、b43-tools、bash、bc、bison、bzip2、cbootimage、cbootimage-configs、ccache、cmake、coreutils、cpio、dosfstools、dwarves、e2fsprogs、elftosb、elfutils、erofs-utils、expat、fakeroot、findutils、firmware-utils、flex、flock、genext2fs、gengetopt、gmp、gnulib、gptfdisk、isl、libdeflate、liblzo、libressl、libtool、llvm-bpf、lz4、lzma、lzma-old、lzop、m4、make-ext4fs、meson、missing-macros、mkimage、mklibs、mold、mpc、mpfr、mtd-utils、mtools、ninja、padjffs2、patch、patch-image、patchelf、pkgconf、popt、quilt、sdimage、sed、sparse、squashfs3-lzma、squashfs4、sstrip、tar、util-linux、xxhash、xz、yafut、zip、zlib、zstd

---

## 7. 目标平台模块（target/）

### 7.1 [target/Makefile](file:///workspace/target/Makefile) — 目标系统协调

**职责**：协调目标系统构建，包括内核、SDK、Image Builder、预编译工具链、LLVM-BPF。

**builddirs**：`linux sdk imagebuilder toolchain llvm-bpf`

**builddirs-install**（条件性）：

- `linux`（始终）
- `$(if $(CONFIG_SDK),sdk)`
- `$(if $(CONFIG_IB),imagebuilder)`
- `$(if $(CONFIG_MAKE_TOOLCHAIN),toolchain)`
- `$(if $(CONFIG_SDK_LLVM_BPF),llvm-bpf)`

**依赖**：`sdk/install` 和 `imagebuilder/install` 都依赖 `linux/install`

### 7.2 [target/Config.in](file:///workspace/target/Config.in) — 硬件特性与架构

**硬件特性**（bool 标志，由各 subtarget 选择）：

- 计算：`HAS_FPU`、`HAS_SPE_FPU`、`HAS_MIPS16`、`HAS_DT_OVERLAY_SUPPORT`、`HAS_TESTING_KERNEL`、`BIG_ENDIAN`、`NOMMU`、`ARCH_64BIT`
- I/O：`AUDIO_SUPPORT`、`GPIO_SUPPORT`、`PCI_SUPPORT`、`PCIE_SUPPORT`、`PCMCIA_SUPPORT`、`PINCTRL_SUPPORT`、`PWM_SUPPORT`、`USB_SUPPORT`、`USB_GADGET_SUPPORT`、`RTC_SUPPORT`、`RFKILL_SUPPORT`、`VIRTIO_SUPPORT`
- 虚拟化：`USES_PM`、`USES_DEVICETREE`、`USES_INITRAMFS`、`USES_SEPARATE_INITRAMFS`
- 文件系统：`USES_SQUASHFS`、`USES_JFFS2`、`USES_JFFS2_NAND`、`USES_EXT4`、`USES_EROFS`、`USES_TARGZ`、`USES_CPIOGZ`、`USES_UBIFS`、`USES_MINOR`、`USES_ROOTFS_PART`、`USES_BOOT_PART`
- 存储：`LOW_MEMORY_FOOTPRINT`、`SMALL_FLASH`、`EMMC_SUPPORT`、`NAND_SUPPORT`、`LEGACY_SDCARD_SUPPORT`、`REGULATOR_SUPPORT`

**架构选择**：aarch64、aarch64_be、arm、armeb、arm_v6、arm_v7、i386、i686、loongarch64、mips、mipsel、mips64、mips64el、powerpc、powerpc64、riscv64、x86_64

### 7.3 [target/linux/](file:///workspace/target/linux/) — Linux 内核目标支持

**主 Makefile**（[target/linux/Makefile](file:///workspace/target/linux/Makefile)）：

- 极简，仅 `include target.mk`、`default-packages.mk`
- `export TARGET_BUILD=1`
- 所有目标都委托给 `$(firstword $(wildcard feeds/$(BOARD) $(BOARD)))` 目录，支持 feeds 中的 board

**subtarget Makefile 模式**：

```makefile
ARCH:=<arch>
BOARD:=<board>
BOARDNAME:=<description>
CPU_TYPE:=<cpu>
CPU_SUBTYPE:=<subtype>  (可选)
SUBTARGETS:=<sub1> <sub2> ...
FEATURES:=<feature1> <feature2> ...
KERNEL_PATCHVER:=<version>
KERNELNAME:=<kernel image names>  (可选)
include $(INCLUDE_DIR)/target.mk
DEFAULT_PACKAGES += <packages>
$(eval $(call BuildTarget))
```

**具体 subtarget 示例**：

| subtarget | ARCH | CPU_TYPE | KERNEL_PATCHVER | SUBTARGETS | FEATURES | DEFAULT_PACKAGES 特点 |
|-----------|------|----------|-----------------|------------|----------|----------------------|
| [ipq806x](file:///workspace/target/linux/ipq806x/) | arm | cortex-a15 (neon-vfpv4) | 6.12 | generic, chromium | squashfs fpu ramdisk | ath10k-ct, wpad-openssl, USB, ahci, cpufreq |
| [qualcommax](file:///workspace/target/linux/qualcommax/) | aarch64 | cortex-a53 | 6.18 | ipq807x, ipq60xx, ipq50xx | squashfs ramdisk fpu nand rtc emmc | 大量 qca-nss-* 驱动（NSS 网络加速）、ath11k、DSA |
| [mediatek](file:///workspace/target/linux/mediatek/) | arm | - | 6.18 | filogic, mt7622, mt7623, mt7629 | dt-overlay emmc fpu gpio nand pci pcie rootfs-part separate_ramdisk squashfs usb | leds-gpio, gpio-button-hotplug |
| [ath79](file:///workspace/target/linux/ath79/) | mips | 24kc | 6.12 (testing 6.18) | generic, mikrotik, nand, tiny | ramdisk squashfs usbgadget | gpio-button-hotplug, swconfig, ath9k, uboot-envtools |

**subtarget 目录结构**（以 ath79 为参考）：

- `Makefile`：平台定义
- `config-6.12`、`config-6.18`：内核配置
- `modules.mk`：目标特定内核模块
- `dts/`：设备树源文件
- `image/`：镜像构建
  - `Makefile`：定义 `Build/loader-*`、`Device/*` 宏
  - `common-<vendor>.mk`：厂商通用镜像规则
  - `generic.mk`、`tiny.mk`、`nand.mk`、`mikrotik.mk`：按子类型组织
- `patches-6.18/`：内核补丁（编号体系：100-900+，按子系统分类）
- `tiny/`、`generic/`：子类型特定配置
- `profiles/`：设备 profile 定义

**generic/ 目录**：跨平台共享的内核补丁

- `backport-6.12/`、`backport-6.18/`：内核回迁补丁
- `pending-6.12/`、`hack-6.12/`、`pending-6.18/`、`hack-6.18/`：待处理和 hack 补丁
- `config-6.12`、`config-6.18`、`config-filter`、`kernel-6.12`、`kernel-6.18`：通用内核配置和版本定义

**支持的平台**（约 40 个）：airoha、apm821xx、armsr、at91、ath79、bcm27xx、bcm47xx、bcm4908、bcm53xx、bmips、d1、econet、gemini、imx、ipq40xx、ipq806x、ixp4xx、kirkwood、lantiq、layerscape、loongarch64、malta、mediatek、microchipsw、mpc85xx、mvebu、mxs、octeon、omap、pistachio、qoriq、qualcommax、qualcommbe、ramips、realtek、rockchip、sifiveu、siflower、starfive、stm32、sunxi、tegra、uml、x86、zynq

### 7.4 [target/imagebuilder/](file:///workspace/target/imagebuilder/) — Image Builder

**职责**：生成 Image Builder（IB）tarball，允许用户无需编译即可生成自定义镜像。

**Config.in**：

- `IB`：构建 Image Builder（默认 BUILDBOT）
- `IB_STANDALONE`：包含包仓库（非 BUILDBOT 时默认 y；禁用则仅嵌入 toolchain 和 kmod，其他 ipk 从在线仓库获取）

**构建步骤**（生成 `$(BIN_DIR)/$(IB_NAME).tar.zst`）：

1. 创建目录结构
2. 复制 .config、INCLUDE_DIR、SCRIPT_DIR、rules.mk、files/Makefile、.targetinfo、.packageinfo
3. APK 路径：`FeedSourcesAppendAPK` 生成 repositories；OPKG 路径：`FeedSourcesAppendOPKG` 生成 repositories.conf
4. 复制 base-files/libc/kernel 包到 packages/
5. 签名处理：复制 opkg/apk keys
6. 复制 target/linux/Makefile、generic、BOARD 目录
7. 复制 grub、kernel build 产物
8. 复制 dtc、DTS 文件
9. SED 替换 version.mk 和 kernel.mk 中的版本信息
10. 复制 STAGING_DIR_IMAGE、staging_dir/host/bin
11. bundle-libraries、复制 libfakeroot、rstrip
12. zstd 压缩（`-T0 --ultra -20`），使用 SOURCE_DATE_EPOCH 作为 mtime

**files/Makefile**（IB 内部 Makefile）：

- 提供命令：`help`、`info`、`clean`、`image`、`manifest`、`package_whatdepends`、`package_depends`
- `image` 参数：PROFILE、PACKAGES、FILES、BIN_DIR、EXTRA_IMAGE_NAME、DISABLED_SERVICES、ADD_LOCAL_KEY、ROOTFS_PARTSIZE、ROOTFS_FILESYSTEM
- 同时支持 OPKG 和 APK 包管理器

### 7.5 [target/sdk/](file:///workspace/target/sdk/) — SDK 生成

**职责**：生成 SDK tarball，包含预编译工具链，用于开发和测试包。

**Config.in**：

- `SDK`：构建 OpenWrt SDK（默认 BUILDBOT）
- `SDK_LLVM_BPF`：构建 LLVM-BPF 工具链 tarball

**关键内容**：

- `SDK_NAME`：包含版本/board/subtarget/gcc 版本/host 信息的文件名
- `BASE_FEED`：自动检测 git/svn 来源，回退到 `https://github.com/immortalwrt/immortalwrt.git`
- `EXCLUDE_DIRS`：排除 stamp/stampfiles/man/info 等
- `KERNEL_FILES_ARCH`、`KERNEL_FILES_BASE`：精简的内核源文件

**files/Makefile**（SDK 内部 Makefile）：

- `SDK:=1`，`world` 为默认目标
- `clean`：git clean staging_dir/build_dir/bin_dir
- `dirclean`：git reset --hard + git clean + rm feeds/
- `prereq`：依赖 package/stamp-prereq
- `world`：prepare → package/stamp-compile → package/index

### 7.6 [target/toolchain/](file:///workspace/target/toolchain/) — 预编译工具链打包

**职责**：将构建好的工具链打包为可分发的 tarball。

**Config.in**：`MAKE_TOOLCHAIN`（默认 BUILDBOT，依赖 `!EXTERNAL_TOOLCHAIN`）

**构建步骤**：

1. tar 从 staging_dir 复制 toolchain 目录
2. 复制 LICENSES、COPYING、files/README.TOOLCHAIN
3. 复制 `files/wrapper.sh` 为 `$(REAL_GNU_TARGET_NAME)-wrapper.sh`
4. 为 cc/gcc/g++/c++/cpp/ld/as 创建 wrapper：原二进制重命名为 `.bin`，符号链接到 wrapper.sh
5. strip 所有 bin/libexec
6. 生成 version.mk
7. zstd 压缩

**files/wrapper.sh**：

- 解析工具名获取 TOOLCHAIN_PLATFORM 和 BINARY
- 设置 TOOLCHAIN_SYSROOT
- 根据 libc 类型设置 `--sysroot` 和 `-rpath-link`
- 对 cc/gcc/g++/c++/cpp 添加 GCC_SYSROOT_FLAGS + CFLAGS
- 对 ld 添加 LD_SYSROOT_FLAGS
- 对 as 添加 ASFLAGS
- 调用 `$(TARGET_TOOLCHAIN_TRIPLET)-$BINARY.bin`

### 7.7 [target/llvm-bpf/](file:///workspace/target/llvm-bpf/) — LLVM BPF 工具链

**职责**：将预编译的 LLVM BPF 工具链打包为 tarball，用于 eBPF 开发。

- `LLVM_VERSION := $(shell cat $(STAGING_DIR_HOST)/llvm-bpf/.llvm-version)`
- 从 `$(STAGING_DIR_HOST)` 打包 `llvm-bpf` 和 `$(LLVM_BPF_PREFIX)` 两个目录
- zstd 压缩（`-T0 --ultra -20`）

---

## 8. 软件包模块（package/）

### 8.1 [package/Makefile](file:///workspace/Makefile) — 主包构建协调器

**职责**：协调所有子目录下包的编译、安装、索引生成。

**关键机制**：

- 设置 `curdir:=package`，包含 `feeds.mk` 和 `rootfs.mk`
- 维护三个包列表：`package-y`（编译）、`package-m`（模块化）、`package-`（不编译），并强制加入 `kernel/linux`
- `PACKAGE_INSTALL_FILES` 宏处理包变体（variants），支持 default-variant 机制
- 通过 `IGNORE_ERRORS` 支持忽略特定错误级别（n/m/y）继续构建

**关键目标**：

| 目标 | 说明 |
|------|------|
| `cleanup` | 清除 `STAGING_DIR_ROOT` |
| `merge` | 将所有子目录的 `.apk`/`.ipk` 包符号链接到 `PACKAGE_DIR_ALL` |
| `merge-index` | 生成包索引（APK 用 `apk mkndx`，OPKG 用 `ipkg-make-index.sh`），支持签名 |
| `install` | 将包安装到 `TARGET_DIR`，APK 路径用 `apk add`，OPKG 路径用 `opkg install`，然后调用 `prepare_rootfs` |
| `index` | 为每个 `PACKAGE_SUBDIRS` 生成 `Packages`/`packages.adb` 索引、`index.json`、可选的 CycloneDX SBOM，最后生成 sha256 校验 |

**双包管理器支持**：通过 `CONFIG_USE_APK` 切换 APK（Alpine Package Keeper）和 OPKG 两套流程。

### 8.2 [package/base-files/Makefile](file:///workspace/package/base-files/Makefile) — 基础文件系统包

**职责**：构建系统的基础根文件系统，包含系统脚本、配置文件、目录结构。

**包定义**：

- `PKG_NAME:=base-files`，`PKG_FLAGS:=nonshared`（不共享，每个目标独立构建）
- `PKG_RELEASE:=$(COMMITCOUNT)`（基于 git 提交计数）
- 依赖：`netifd`、`libc`、`jsonfilter`、`usign`、`openwrt-keyring`、`fstables`、`fwtool`、`procd`、`busybox` 等

**关键函数**：

- `ImageConfigOptions`：生成 `/lib/preinit/00_preinit.conf`，写入 preinit 网络配置
- `Build/Configure`：生成签名密钥（usign 生成密钥对，ucert 签发证书）
- `Package/base-files/install`：核心安装逻辑
  - 复制 `./files/*`、平台特定 base-files
  - 用 `VERSION_SED_SCRIPT` 注入版本信息到 banner、device_info、openwrt_release、os-release
  - 创建标准目录结构（dev、etc、proc、sys、tmp、var、usr、www、overlay 等）
  - 建立符号链接（`/lib` → `/proc/mounts` 作 mtab，`/var` → `/tmp` 等）
  - 配置 APK 仓库（`/etc/apk/repositories.d/distfeeds.list`）或 OPKG 仓库
  - 处理按钮自定义（failsafe/power/reboot/reset/rfkill）

### 8.3 [package/base-files/image-config.in](file:///workspace/package/base-files/image-config.in) — 镜像配置

**配置分组**：

- `EXTRA_IMAGE_NAME`：额外镜像文件名标识
- `PREINITOPT`：failsafe 开关、超时、网络消息抑制、preinit 网络接口/IP/netmask/broadcast
- `INITOPT`：PATH、环境变量、init 命令、stderr 抑制
- `VERSIONOPT`：发行版名称（默认 `ImmortalWrt`）、版本号、版本代码、仓库 URL（支持 `%R/%V/%v/%C/%c/%D/%d/%T/%S/%A/%t/%M/%P/%h` 占位符）、主页/制造商/产品/硬件版本
- `TARGET_BUTTON_CUSTOMIZATION`：禁用各类按钮处理
- `PER_FEED_REPO`：分离的 feed 仓库（默认开启）

### 8.4 package/ 子目录组织

包按类别组织：

| 目录 | 说明 |
|------|------|
| `base-files/` | 基础文件系统 |
| `boot/` | 引导加载器（U-Boot 各平台、ATF、grub2、opensbi 等） |
| `devel/` | 开发工具（gdb、strace、valgrind、perf 等） |
| `emortal/` | ImmortalWrt 特有（autocore、automount、cpufreq、default-settings） |
| `firmware/` | 固件（linux-firmware、ath10k/11k、intel-microcode 等） |
| `kernel/` | 内核模块（mac80211、mt76、r8168、rtl8812au 等） |
| `libs/` | 库（openssl、mbedtls、wolfssl、zlib、libubox 等） |
| `qca-nss/` | 高通 NSS 加速相关 |
| `system/` | 系统组件（procd、ubus、uci、opkg、apk、fstables、mtd 等） |
| `utils/` | 工具（busybox、util-linux、lua、ucode 等） |

---

## 9. 脚本模块（scripts/）

### 9.1 [scripts/feeds](file:///workspace/scripts/feeds) — Feed 管理脚本（Perl）

**职责**：管理外部软件源（feeds）的更新、安装、卸载、搜索。

**核心命令**：

- `update [-a|-i|-f|-r|-s] [feed...]`：更新 feed 源码并重建索引
- `install [-a|-p|-d|-f] <package...>`：安装 feed 包（创建符号链接到 `package/feeds/<feed>/<pkg>`）
- `uninstall [-a] <package...>`：卸载 feed 包
- `list [-n|-s|-r|-d|-f]`：列出 feed 及内容
- `search [-r <feed>] <substring...>`：搜索包
- `clean`：清除所有 feed 数据
- `feed_config`：生成 feed 的 Kconfig 配置

**支持的源类型**：

- `src-git`：浅克隆（`--depth 1`），支持分支、commit、rebase、autostash、force 更新、子模块
- `src-git-full`：完整克隆（`--filter=blob:none`）
- `src-svn`、`src-bzr`、`src-hg`、`src-darcs`：其他 VCS
- `src-cpy`、`src-link`：本地路径复制/链接

### 9.2 [scripts/download.pl](file:///workspace/scripts/download.pl) — 下载脚本（Perl）

**职责**：从多个镜像下载源码包并校验哈希。

**镜像系统**：

- 本地镜像：`localmirrors` 文件、`CONFIG_LOCALMIRROR`、`DOWNLOAD_MIRROR` 环境变量
- 项目镜像（`projectsmirrors.json`）：`@SF`、`@OPENWRT`、`@IMMORTALWRT`、`@DEBIAN`、`@APACHE`、`@GITHUB`、`@GNU`、`@SAVANNAH`、`@KERNEL`、`@GNOME`
- `@GITHUB` 特殊处理：添加 jsDelivr 和 gitmirror 镜像
- `@KERNEL` 特殊处理：根据文件名识别 testing/longterm 子目录

**下载工具选择**：`select_tool` 优先 curl，回退 wget；支持自定义 `DOWNLOAD_TOOL_CUSTOM`；aria2c 支持多连接下载

**哈希校验**：根据哈希长度自动选择算法（64 字符 → SHA256，32 字符 → MD5），通过 `MKHASH` 环境变量调用

### 9.3 [scripts/functions.sh](file:///workspace/scripts/functions.sh) — 通用 shell 函数

**职责**：提供文件系统类型检测的共享函数。

**函数**：

- `get_magic_word`：读取文件头 4 字节（十六进制）
- `get_post_padding_word`：读取 JFFS2 末尾标记
- `get_fs_type`：根据魔数识别 `ubifs`（0x3118）、`squashfs`（0x68737173）、`squashfs-jffs2`、`unknown`
- `round_up`：向上对齐计算

### 9.4 [scripts/getver.sh](file:///workspace/scripts/getver.sh) — 版本获取

**职责**：获取当前构建的版本标识符。

**策略链**：`try_version || try_git || try_hg || REV="unknown"`

- `try_version`：读取 `version` 文件
- `try_git`：基于 `REBOOT` 基准 commit 计算提交数，生成 `r<数>-<短hash>` 格式
- `try_hg`：Mercurial 支持

### 9.5 [scripts/diffconfig.sh](file:///workspace/scripts/diffconfig.sh) — 配置差异生成

**职责**：生成最小化的 `.config` 差异，便于复现构建。

**流程**：

1. 构建 `scripts/config/conf`
2. 提取关键配置到 `.diffconfig.head`
3. 两轮 `config/conf --defconfig` + `kconfig.pl` 比较，提取与默认配置的差异
4. 输出最小配置

### 9.6 [scripts/ipkg-build](file:///workspace/scripts/ipkg-build) — IPK 包构建器

**职责**：将包目录打包成 `.ipk` 文件（OPKG 包格式）。

**流程**：

1. 验证包结构（`CONTROL/control` 必需字段）
2. 校验包名合法性
3. 解析 `conffiles`、`file_modes`
4. 创建 `data.tar.gz`（包内容，支持 `SOURCE_DATE_EPOCH` 可重现构建）
5. 创建 `control.tar.gz`（CONTROL 目录内容）
6. 创建 `debian-binary`（"2.0"）
7. 打包成 `${pkg}_${version}_${arch}.ipk`

### 9.7 [scripts/ext-toolchain.sh](file:///workspace/scripts/ext-toolchain.sh) — 外部工具链管理

**职责**：支持使用预构建的外部工具链。

**核心功能**：

- **探测**：`probe_cc`/`probe_cxx`/`probe_cpp`、`probe_libc`、`find_gcc_version`
- **特性测试**：`test_feature` 测试 c/c++/softfloat/lfs/ipv6/rpc/locale/wchar/threads
- **查找**：`find_libs`、`find_bins`
- **包装**：`wrap_bins` 为编译器/链接器创建包装脚本
- **配置生成**：`print_config` 分析工具链并生成对应的 `.config`

### 9.8 [scripts/metadata.pm](file:///workspace/scripts/metadata.pm) — Perl 元数据模块

**职责**：解析包和目标元数据，提供共享数据结构。

**导出的全局变量**：`%package`、`%vpackage`、`%srcpackage`、`%category`、`%overrides`、`%usernames`、`%groupnames`

**核心函数**：

- `parse_package_metadata($file)`：解析 `.packageinfo` 文件
- `parse_target_metadata($file)`：解析 `.targetinfo`
- `parse_package_manifest_metadata($file)`：解析 `Packages` 清单
- `parse_package_metadata_usergroup`：解析用户/组规格

### 9.9 [scripts/package-metadata.pl](file:///workspace/scripts/package-metadata.pl) — 包元数据处理

**职责**：将包元数据转换为多种输出格式。

**命令**：

- `mk`：生成 makefile 格式的包依赖关系
- `config`：生成 Kconfig 配置树
- `kconfig`：生成内核配置覆盖
- `source`：输出包源 URL 信息
- `pkgaux`：输出包辅助变量
- `pkgmanifestjson`：生成 JSON 格式包清单
- `imgcyclonedxsbom`/`pkgcyclonedxsbom`：生成 CycloneDX 1.4 SBOM
- `license`/`licensefull`：输出许可证信息
- `usergroup`：输出用户/组分配列表
- `version_filter`：版本过滤

### 9.10 [scripts/target-metadata.pl](file:///workspace/scripts/target-metadata.pl) — 目标元数据处理

**职责**：将目标（硬件平台）元数据转换为 Kconfig 和 makefile 格式。

**命令**：

- `config`：生成目标 Kconfig（`choice "Target System"`、`choice "Subtarget"`、`choice "Target Profile"`、`TARGET_DEVICE_*` 配置等）
- `profile_mk`：生成指定目标的 profile makefile 变量

### 9.11 [scripts/config/](file:///workspace/scripts/config/) — Kconfig 配置工具

**职责**：提供 Linux 内核风格的配置工具。

**工具**：

- `conf`（[conf.c](file:///workspace/scripts/config/conf.c)）：命令行配置，支持 `defconfig`、`oldconfig`、`olddefconfig`、`savedefconfig`
- `mconf`（[mconf.c](file:///workspace/scripts/config/mconf.c)）：基于 ncurses/lxdialog 的菜单配置（`menuconfig`）
- `nconf`（[nconf.c](file:///workspace/scripts/config/nconf.c) + `nconf.gui.c`）：基于 ncurses 的新式菜单配置（`nconfig`）
- `qconf`（[qconf.cc](file:///workspace/scripts/config/qconf.cc)）：基于 Qt 的图形配置（`xconfig`）

**核心组件**：

- `expr.c`/`expr.h`：表达式求值
- `symbol.c`：符号管理
- `menu.c`：菜单树管理
- `confdata.c`：配置文件读写
- `lexer.l`/`parser.y`：Kconfig 语法的词法/语法分析
- `preprocess.c`：预处理

### 9.12 [scripts/checkpatch.pl](file:///workspace/scripts/checkpatch.pl) — 补丁检查器

**职责**：检查补丁的编码风格，源自 Linux 内核（版本 `0.32-openwrt`），适配 OpenWrt 树。

**特性**：

- 检查行长度（默认 100）、签名、编码风格
- 使用 `spelling.txt`、`const_structs.checkpatch`
- 支持 `--fix`/`--fix-inplace` 自动修复
- 支持 git 模式检查提交

### 9.13 [scripts/strip-kmod.sh](file:///workspace/scripts/strip-kmod.sh) — 内核模块精简器

**职责**：精简内核模块（.ko）以减小体积。

**流程**：

1. 用 `objcopy` 移除无用节（`.comment`、`.pdr`、`.mdebug.abi32`、`.gnu.attributes`、`.reginfo`、`.MIPS.abiflags`、`.note.GNU-stack`）
2. 根据 `KEEP_SYMBOLS` 选择剥离策略
3. 根据 `KEEP_BUILD_ID` 决定是否移除 `.note.gnu.build-id`
4. 若未设置 `NO_RENAME`：用 `nm` 提取符号，`objcopy --redefine-sym` 重命名全局符号避免模块间符号冲突

### 9.14 其他重要脚本

| 脚本 | 说明 |
|------|------|
| [scripts/rstrip.sh](file:///workspace/scripts/rstrip.sh) | 剥离目标二进制文件的调试信息 |
| [scripts/bundle-libraries.sh](file:///workspace/scripts/bundle-libraries.sh) | 将库打包到二进制旁，调整 RPATH |
| [scripts/symlink-tree.sh](file:///workspace/scripts/symlink-tree.sh) | 符号链接目录树 |
| [scripts/mkits.sh](file:///workspace/scripts/mkits.sh) | 生成 FIT image ITS 文件 |
| [scripts/combined-image.sh](file:///workspace/scripts/combined-image.sh) | 合并内核和 rootfs 镜像 |
| [scripts/sysupgrade-tar.sh](file:///workspace/scripts/sysupgrade-tar.sh) | 生成 sysupgrade 镜像 |
| [scripts/ubinize-image.sh](file:///workspace/scripts/ubinize-image.sh) | 生成 UBI 镜像 |
| [scripts/sign_images.sh](file:///workspace/scripts/sign_images.sh) | 签名镜像 |
| [scripts/mkhash.c](file:///workspace/scripts/mkhash.c) | 哈希计算工具（C 源码） |
| [scripts/kconfig.pl](file:///workspace/scripts/kconfig.pl) | Kconfig 配置比较工具 |
| [scripts/timestamp.pl](file:///workspace/scripts/timestamp.pl) | 时间戳处理 |
| [scripts/get_source_date_epoch.sh](file:///workspace/scripts/get_source_date_epoch.sh) | 获取 SOURCE_DATE_EPOCH |
| [scripts/make-sbom.py](file:///workspace/scripts/make-sbom.py) | 生成 SBOM |
| [scripts/json_add_image_info.py](file:///workspace/scripts/json_add_image_info.py) | 生成镜像信息的 JSON |
| [scripts/json_overview_image_info.py](file:///workspace/scripts/json_overview_image_info.py) | 生成镜像概览 JSON |
| [scripts/dl_cleanup.py](file:///workspace/scripts/dl_cleanup.py) | 下载目录清理 |
| [scripts/dl_github_archive.py](file:///workspace/scripts/dl_github_archive.py) | GitHub 归档下载 |
| [scripts/patch-kernel.sh](file:///workspace/scripts/patch-kernel.sh) | 内核补丁应用 |
| [scripts/flashing/](file:///workspace/scripts/flashing/) | 各平台刷机脚本 |

---

## 10. 配置模块（config/）

### 10.1 [config/Config-build.in](file:///workspace/config/Config-build.in) — 全局构建设置

**职责**：定义 "Global build settings" 菜单，控制整个发行版的构建行为、安全加固、二进制处理方式。

**关键配置项**：

- **构建模式**：`EXPERIMENTAL`、`BUILDBOT`、`ALL`/`ALL_KMODS`/`ALL_NONSHARED`
- **包管理**：`USE_APK`（默认 y）、`SIGNED_PACKAGES`/`SIGNATURE_CHECK`/`SIGN_EACH_PACKAGE`、`DOWNLOAD_CHECK_CERTIFICATE`
- **特性开关**：`TESTING_KERNEL`、`BUILD_PATENTED`、`BUILD_NLS`、`IPV6`、`SHADOW_PASSWORDS`
- **二进制处理**：剥离方式 choice（`NO_STRIP`/`USE_STRIP`/`USE_SSTRIP`）、`STRIP_ARGS`、`STRIP_KERNEL_EXPORTS`、`USE_MKLIBS`、`DEBUG`、`USE_GC_SECTIONS`、`USE_LTO`、`MOLD`/`USE_MOLD`
- **安全加固 choice 组**：
  - 用户态 PIE：`PKG_ASLR_PIE_NONE/REGULAR/ALL`
  - 用户态 SSP：`PKG_CC_STACKPROTECTOR_NONE/REGULAR/STRONG/ALL`
  - 内核态 SSP：`KERNEL_CC_STACKPROTECTOR_*`
  - `_FORTIFY_SOURCE`：`PKG_FORTIFY_SOURCE_NONE/1/2/3`
  - RELRO：`PKG_RELRO_NONE/PARTIAL/FULL`
  - `PKG_DT_RELR`、`PKG_FANALYZER`、`PKG_CHECK_FORMAT_SECURITY`
- **安全子系统**：`SELINUX`、`TARGET_ROOTFS_SECURITY_LABELS`、`USE_SECCOMP`
- **JSON/SBOM**：`JSON_OVERVIEW_IMAGE_INFO`、`JSON_CYCLONEDX_SBOM`
- **调试信息**：`REPRODUCIBLE_DEBUG_INFO`、`COLLECT_KERNEL_DEBUG`

### 10.2 [config/Config-devel.in](file:///workspace/config/Config-devel.in) — 开发者高级选项

**职责**：定义 `menuconfig DEVEL`，仅在开启 DEVEL 时可见。

**关键配置项**：

- `BROKEN`：显示损坏的平台/包/设备
- 路径覆盖：`BINARY_FOLDER`、`DOWNLOAD_FOLDER`、`LOCALMIRROR`、`BUILD_SUFFIX`、`TARGET_ROOTFS_DIR`
- 下载工具：`DOWNLOAD_TOOL_CUSTOM`
- 构建行为：`AUTOREBUILD`、`AUTOREMOVE`、`BUILD_ALL_HOST_TOOLS`、`SRC_TREE_OVERRIDE`
- 主机工具优化：`HOST_FLAGS_OPT`、`HOST_TOOLS_STRIP`、`HOST_FLAGS_STRIP`、`HOST_EXTRA_CFLAGS/CXXFLAGS/CPPFLAGS/LDFLAGS`
- ccache：`CCACHE`、`CCACHE_DIR`
- 内核源码：`KERNEL_CFLAGS`、`EXTERNAL_KERNEL_TREE`、`KERNEL_GIT_CLONE_URI`、`KERNEL_GIT_LOCAL_REPOSITORY`、`KERNEL_GIT_REF`、`KERNEL_GIT_MIRROR_HASH`
- 日志：`BUILD_LOG`、`BUILD_LOG_DIR`
- `EXTRA_OPTIMIZATION`：额外目标无关优化标志

### 10.3 [config/Config-images.in](file:///workspace/config/Config-images.in) — 目标镜像配置

**职责**：定义 "Target Images" 菜单，控制 rootfs 类型、压缩、分区、虚拟机/光驱镜像生成。

**关键配置项**：

- **initramfs**：`TARGET_ROOTFS_INITRAMFS`、压缩 choice（NONE/GZIP/BZIP2/LZMA/LZO/LZ4/XZ/ZSTD）、`EXTERNAL_CPIO`、`TARGET_INITRAMFS_FORCE`、`TARGET_ROOTFS_INITRAMFS_SEPARATE`
- **rootfs 归档**：`TARGET_ROOTFS_CPIOGZ`、`TARGET_ROOTFS_TARGZ`
- **rootfs 镜像类型**：
  - `TARGET_ROOTFS_EROFS` + `TARGET_EROFS_PCLUSTER_SIZE`
  - `TARGET_ROOTFS_EXT4FS` + `TARGET_EXT4_RESERVED_PCT`、块大小 choice、`TARGET_EXT4_JOURNAL`
  - `TARGET_ROOTFS_JFFS2` / `TARGET_ROOTFS_JFFS2_NAND`
  - `TARGET_ROOTFS_SQUASHFS` + `TARGET_SQUASHFS_BLOCK_SIZE`
  - `TARGET_ROOTFS_UBIFS` + 压缩 choice、`TARGET_UBIFS_FREE_SPACE_FIXUP`、`TARGET_UBIFS_JOURNAL_SIZE`
- **x86/虚拟化镜像**：`GRUB_IMAGES`、`GRUB_EFI_IMAGES`、`GRUB_CONSOLE/BAUDRATE/FLOWCONTROL/BOOTOPTS/TIMEOUT/TITLE`（默认 "ImmortalWRT"）、`ISO_IMAGES`、`QCOW2_IMAGES`、`VDI_IMAGES`、`VMDK_IMAGES`、`VHDX_IMAGES`、`ONIE_INSTALLER_IMAGES`
- **串口/分区**：`TARGET_SERIAL`、`TARGET_IMAGES_GZIP`、`TARGET_KERNEL_PARTSIZE`、`TARGET_ROOTFS_PARTSIZE`、`TARGET_ROOTFS_PARTNAME`

### 10.4 [config/Config-ipq.in](file:///workspace/config/Config-ipq.in) — Qualcomm IPQ 专属选项

**职责**：定义 "Qualcomm IPQ Options" 菜单，针对高通 IPQ 平台的内存与 SKB 回收调优。

**关键配置项**：

- `KERNEL_IPQ_MEM_PROFILE` choice：1024 / 512 / 256（默认 1024）
- `KERNEL_SKB_RECYCLER`：通用 SKB 回收
- `KERNEL_SKB_RECYCLE_SIZE` choice：2304 / 1856
- `KERNEL_SKB_RECYCLER_MULTI_CPU`：跨 CPU 回收
- `KERNEL_SKB_RECYCLER_PREALLOC` + `KERNEL_SKB_RECYCLE_MAX_PREALLOC_SKBS`（默认 16384，约 64MB）
- `KERNEL_SKB_FIXED_SIZE_2K`
- `KERNEL_ALLOC_SKB_PAGE_FRAG_DISABLE`

### 10.5 [config/Config-kernel.in](file:///workspace/config/Config-kernel.in) — 内核功能配置

**职责**：定义内核侧功能开关，是整个构建系统中最大的 Kconfig 文件（1694 行）。

**关键配置项（按类别）**：

- **构建标识**：`KERNEL_BUILD_USER`、`KERNEL_BUILD_DOMAIN`
- **基础特性**：`KERNEL_PRINTK`、`KERNEL_SWAP`、`KERNEL_PROC_STRIPPED`、`KERNEL_DEBUG_FS`、`KERNEL_NR_CPUS`
- **性能/PMU**：`KERNEL_PERF_EVENTS`、`KERNEL_PROFILING`
- **Sanitizer**：`KERNEL_UBSAN` 系列、`KERNEL_KASAN`（Generic/SW_TAGS/HW_TAGS）、`KERNEL_KCOV` 系列
- **调试**：`KERNEL_DEBUG_INFO`、`KERNEL_DEBUG_INFO_BTF`、`KERNEL_KPROBES`/`KERNEL_KPROBE_EVENTS`、`KERNEL_SOFTLOCKUP_DETECTOR`/`KERNEL_HARDLOCKUP_DETECTOR`/`KERNEL_DETECT_HUNG_TASK`
- **ftrace 系列**：`KERNEL_FTRACE` 及其下 SYSCALLS/DEFAULT_TRACERS/FUNCTION_TRACER/FUNCTION_GRAPH_TRACER/DYNAMIC_FTRACE 等
- **IO/内存**：`KERNEL_AIO`、`KERNEL_IO_URING`、`KERNEL_FHANDLE`、`KERNEL_FANOTIFY`、`KERNEL_TRANSPARENT_HUGEPAGE`
- **cgroup**：FREEZER/DEVICE/HUGETLB/PIDS/RDMA/BPF/CPUSETS/CPUACCT/MEMCG/CGROUP_PERF/CGROUP_SCHED/BLK_CGROUP/NET_CLS_CGROUP/NET_CLASSID/NET_PRIO
- **namespace**：UTS/IPC/USER/PID/NET
- **网络**：`KERNEL_IP_MROUTE`、`KERNEL_IPV6`、`KERNEL_MPTCP`、`KERNEL_NF_CONNTRACK_TIMEOUT`
- **文件系统**：`KERNEL_IP_PNP`、`KERNEL_BTRFS_FS`、`KERNEL_EROFS_FS`、`KERNEL_SQUASHFS_FRAGMENT_CACHE_SIZE`、POSIX ACL 菜单
- **安全**：`KERNEL_AUDIT`、`KERNEL_SECURITY`/`KERNEL_SECURITY_NETWORK`、`KERNEL_SECURITY_SELINUX` 系列
- **抢占模型 choice**：`KERNEL_PREEMPT_NONE/VOLUNTARY/PREEMPT/PREEMPT_RT`
- **编译优化 choice**：`KERNEL_CC_OPTIMIZE_FOR_PERFORMANCE`（-O2）/`KERNEL_CC_OPTIMIZE_FOR_SIZE`（-Os）

### 10.6 [config/check-hostcxx.sh](file:///workspace/config/check-hostcxx.sh) 与 [config/check-uname.sh](file:///workspace/config/check-uname.sh)

**check-hostcxx.sh**：在 Kconfig 解析期检测宿主 `g++`/`clang++` 是否满足最低版本要求。被 `Config-build.in` 中 `config MOLD` 使用（参数 `10 2 12` 表示 GCC≥10.2 或 clang≥12）。

**check-uname.sh**：极简的 uname 比较脚本，用于按宿主 OS 启用/禁用选项。

---

## 11. 依赖关系总览

### 11.1 构建依赖图

```
┌─────────────────────────────────────────────────────────────┐
│ 工具链构建（toolchain/）                                      │
│  glibc 路径:                                                 │
│    binutils → gcc/minimal → kernel-headers → glibc/headers   │
│    → gcc/initial → glibc → gcc/final                         │
│  musl 路径:                                                  │
│    binutils → gcc/initial → kernel-headers → musl → gcc/final│
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ 目标系统构建（target/）                                       │
│  ├── linux/ (内核 + 镜像) ← 默认构建                          │
│  ├── sdk/ ← 依赖 linux/install (CONFIG_SDK)                  │
│  ├── imagebuilder/ ← 依赖 linux/install (CONFIG_IB)          │
│  ├── toolchain/ ← 打包工具链 (CONFIG_MAKE_TOOLCHAIN)         │
│  └── llvm-bpf/ ← 打包 LLVM BPF (CONFIG_SDK_LLVM_BPF)         │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ 软件包构建（package/）                                        │
│  ├── base-files/ (基础文件系统)                               │
│  ├── boot/、devel/、emortal/、firmware/、kernel/、libs/、     │
│  │   qca-nss/、system/、utils/                                │
│  └── feeds/ (外部软件源)                                      │
└─────────────────────────────────────────────────────────────┘
```

### 11.2 关键集成点

1. **info.mk 共享**：toolchain 各组件（gcc、glibc、musl）通过 `Host/SetToolchainInfo` 更新 `$(TOOLCHAIN_DIR)/info.mk`，记录 TARGET_CROSS、GCC_VERSION、LIBC_TYPE/URL/VERSION/SO_VERSION，供 target 和 package 构建引用

2. **stampfile 机制**：toolchain 和 target 都使用 `stampfile` 宏管理构建阶段，toolchain 的 compile 戳记依赖 `.gcc_final_installed`，target 的 install 戳记依赖 compile 戳记

3. **subdir 递归**：两个目录都用 `subdir` 宏递归处理 builddirs，依赖关系通过 `:=` 赋值在主 Makefile 中建立

4. **条件构建**：通过 CONFIG_* 变量条件性包含子目录（如 `$(if $(CONFIG_SDK),sdk)`），实现按需构建

5. **包管理器双轨**：imagebuilder 同时支持 OPKG（ipk）和 APK（apk），通过 `CONFIG_USE_APK` 切换

6. **feeds 支持**：target/linux/Makefile 和 imagebuilder 都支持从 `feeds/$(BOARD)` 加载平台，实现外部 feed 扩展

7. **版本一致性**：SDK 和 IB 在打包时通过 SED 替换 version.mk（REVISION、SOURCE_DATE_EPOCH、BASE_FILES_VERSION、LIBC_VERSION、KERNEL_VERSION）和 kernel.mk（LINUX_VERMAGIC），确保分发物版本信息正确

### 11.3 元数据驱动架构

构建系统是元数据驱动的：

1. `include/scan.mk` 扫描所有 `package/*/Makefile` 和 `target/linux/*/Makefile`，生成 `.packageinfo` 和 `.targetinfo`
2. `metadata.pm` 解析这些文件
3. `package-metadata.pl` 和 `target-metadata.pl` 将元数据转换为：
   - Kconfig 配置树（`Config.in`）
   - Makefile 依赖关系（`.packagedeps`）
   - JSON 清单、SBOM 等

---

## 12. 项目运行方式

### 12.1 系统要求

**宿主操作系统**：Linux（推荐 Ubuntu/Debian），macOS 部分支持

**必需工具**（由 `prereq-build.mk` 检查）：

- 编译器：gcc/g++ ≥10
- 构建工具：GNU make ≥4.1、bash、patch、diff、cp、seq、awk、grep、getopt、realpath、stat、gzip、unzip、bzip2、wget、install
- 解释器：perl 5.x、python ≥3.8
- 版本控制：git ≥1.7.12.2、rsync
- 头文件：ncurses.h、argp.h、fts.h、obstack.h、libintl.h
- Perl 模块：Data::Dumper、FindBin、File::Copy、File::Compare、Thread::Queue、IPC::Cmd

### 12.2 基本构建流程

#### 12.2.1 克隆仓库

```bash
git clone https://github.com/VIKINGYFY/immortalwrt.git
cd immortalwrt
```

#### 12.2.2 安装依赖（Ubuntu/Debian 示例）

```bash
sudo apt update
sudo apt install -y build-essential clang flex bison g++ gawk gcc-multilib g++-multilib gettext git libncurses-dev libssl-dev python3-distutils rsync unzip zlib1g-dev
```

#### 12.2.3 更新 feeds（可选）

```bash
./scripts/feeds update -a
./scripts/feeds install -a
```

#### 12.2.4 配置

```bash
make menuconfig
```

在菜单中选择：

- **Target System**：选择目标平台（如 Qualcomm IPQ807x）
- **Subtarget**：选择子目标（如 ipq807x）
- **Target Profile**：选择具体设备
- **其他选项**：包选择、内核配置、镜像格式等

#### 12.2.5 下载源码

```bash
make download V=s
```

#### 12.2.6 构建

```bash
# 首次构建（使用多线程）
make -j$(nproc) V=s

# 仅构建特定包
make package/<package_name>/compile V=s

# 仅构建内核
make target/linux/compile V=s

# 仅构建镜像
make target/linux/install V=s
```

#### 12.2.7 清理

```bash
# 清理构建产物（保留工具链）
make clean

# 清理工具链
make targetclean

# 彻底清理（包括工具链和临时文件）
make dirclean

# 清理 ccache
make cacheclean
```

### 12.3 常用 Make 目标

| 目标 | 说明 |
|------|------|
| `make menuconfig` | 文本菜单配置 |
| `make nconfig` | 新式文本菜单配置 |
| `make xconfig` | Qt 图形配置 |
| `make defconfig` | 默认配置 |
| `make oldconfig` | 更新配置 |
| `make savedefconfig` | 保存最小配置 |
| `make download` | 下载所有源码 |
| `make prereq` | 检查前置条件 |
| `make prepare` | 准备构建（tools + toolchain + buildinfo） |
| `make world` | 完整构建 |
| `make package/index` | 生成包索引 |
| `make checksum` | 生成 sha256 校验和 |
| `make buildinfo` | 生成构建元信息 |
| `make diffconfig` | 生成配置差异 |
| `make kernel_menuconfig` | 内核菜单配置 |
| `make package/<name>/compile` | 编译特定包 |
| `make package/<name>/clean` | 清理特定包 |
| `make package/<name>/refresh` | 刷新补丁 |
| `make target/linux/compile` | 编译内核 |
| `make target/linux/install` | 安装内核镜像 |

### 12.4 高级用法

#### 12.4.1 使用 Image Builder

```bash
# 解压 IB tarball
tar -I zstd -xf immortalwrt-imagebuilder-*-linux-x86_64.tar.zst
cd immortalwrt-imagebuilder-*/

# 查看可用 profile
make info

# 生成镜像
make image PROFILE=generic PACKAGES="pkg1 pkg2 -pkg3" FILES=custom-files/
```

#### 12.4.2 使用 SDK

```bash
# 解压 SDK tarball
tar -I zstd -xf immortalwrt-sdk-*-linux-x86_64.tar.zst
cd immortalwrt-sdk-*/

# 添加 feeds
./scripts/feeds update -a
./scripts/feeds install -a

# 配置
make menuconfig

# 构建
make package/<name>/compile V=s
```

#### 12.4.3 使用外部工具链

```bash
# 探测外部工具链
./scripts/ext-toolchain.sh --toolchain /path/to/toolchain --cflags "-mcpu=cortex-a7"

# 生成 .config
./scripts/ext-toolchain.sh --toolchain /path/to/toolchain --config > .config

# 在 menuconfig 中选择 External toolchain
make menuconfig
```

#### 12.4.4 可重现构建

```bash
# 设置 SOURCE_DATE_EPOCH
export SOURCE_DATE_EPOCH=$(git log -1 --format=%ct)

# 使用 ccache 加速
make menuconfig  # 启用 Developer options → Use ccache

# 构建
make -j$(nproc) V=s
```

#### 12.4.5 调试构建

```bash
# 详细输出
make -j1 V=s

# 启用构建日志
make menuconfig  # 启用 Developer options → Write log files

# 调试特定包
make package/<name>/compile V=s -j1

# 检查补丁风格
./scripts/checkpatch.pl patches/xxx.patch
```

### 12.5 分支说明

- **main**：高通专用，无法编译其他平台，带满血 NSS 驱动
- **owrt**：多平台通用，可编译高通平台，但没有 NSS 驱动

### 12.6 输出产物

构建产物位于 `bin/` 目录：

```
bin/
├── targets/<board>/<subtarget>/         # 镜像文件
│   ├── <prefix>-squashfs-sysupgrade.bin
│   ├── <prefix>-squashfs-factory.bin
│   ├── <prefix>-initramfs-kernel.bin
│   ├── profiles.json
│   └── sha256sums
├── packages/<arch>/                     # 软件包
│   ├── base/
│   ├── <feed>/
│   └── packages.adb / Packages
└── version.buildinfo                    # 版本信息
    config.buildinfo                     # 配置差异
    feeds.buildinfo                      # feeds 信息
```

---

## 13. 关键设计模式与约定

### 13.1 双阶段加载

配置阶段（`OPENWRT_BUILD` 未设）与构建阶段（`=1`）分离，避免配置未完成就执行构建。

### 13.2 Stamp 机制

所有子系统通过 `.prepared`/`.configured`/`.built`/`.installed` stamp 文件跟踪状态，`subdir.mk` 的 `stampfile` 宏统一管理。

### 13.3 递归子目录

`subdir.mk` 通过 `builddirs` 变量声明式地描述构建树，自动生成递归规则。

### 13.4 宏驱动抽象

`BuildPackage`/`HostBuild`/`BuildTarget`/`BuildImage`/`Device`/`KernelPackage`/`Download` 等宏是各子系统的接入点，通过 `eval` + 钩子（Hooks/Pre/Post）实现可定制。

### 13.5 可重现构建

`SOURCE_DATE_EPOCH`、`iremap`、`--numeric-owner --sort=name`、`KBUILD_BUILD_TIMESTAMP` 等贯穿全系统。

### 13.6 并发安全

`locked` 宏（flock）保护共享资源（staging 目录、下载文件）。

### 13.7 多包管理器支持

OPKG 与 APK 双路径，通过 `CONFIG_USE_APK` 切换。

### 13.8 多架构/多目标/多设备

BOARD/SUBTARGET/PROFILE/Device 四层抽象，`target.mk` + `image.mk` 协作处理。

### 13.9 元数据驱动

`scan.mk` + `metadata.pl` 扫描所有包/目标 Makefile 生成 Kconfig 输入，实现声明式配置。

### 13.10 命名约定

- **包名**：`[a-zA-Z0-9_.+-]`
- **Kconfig 符号**：`CONFIG_*`，BOARD/SUBTARGET 归一化（`.`/`-`/`/` → `_`）
- **镜像前缀**：`$(IMG_PREFIX)` 含版本号/board/subtarget
- **工具链命名**：`$(GNU_TARGET_NAME)`、`$(REAL_GNU_TARGET_NAME)`、`$(TARGET_CROSS)`
- **目录布局**：`BUILD_DIR`、`STAGING_DIR`、`BIN_DIR`、`DL_DIR`、`TMP_DIR` 等

---

## 附录：关键文件路径速查

### 顶层文件

- [Makefile](file:///workspace/Makefile) — 顶层构建入口
- [Config.in](file:///workspace/Config.in) — Kconfig 主入口
- [rules.mk](file:///workspace/rules.mk) — 核心变量与宏库
- [feeds.conf.default](file:///workspace/feeds.conf.default) — 默认 feeds 配置

### include/ 核心框架

- [include/toplevel.mk](file:///workspace/include/toplevel.mk) — 顶层构建逻辑
- [include/subdir.mk](file:///workspace/include/subdir.mk) — 子目录递归
- [include/package.mk](file:///workspace/include/package.mk) — 软件包框架
- [include/host-build.mk](file:///workspace/include/host-build.mk) — 主机端构建
- [include/target.mk](file:///workspace/include/target.mk) — 目标平台框架
- [include/image.mk](file:///workspace/include/image.mk) — 镜像生成
- [include/kernel.mk](file:///workspace/include/kernel.mk) — 内核构建
- [include/download.mk](file:///workspace/include/download.mk) — 下载机制
- [include/feeds.mk](file:///workspace/include/feeds.mk) — Feeds 管理
- [include/quilt.mk](file:///workspace/include/quilt.mk) — 补丁管理
- [include/rootfs.mk](file:///workspace/include/rootfs.mk) — rootfs 准备
- [include/depends.mk](file:///workspace/include/depends.mk) — 依赖追踪
- [include/prereq.mk](file:///workspace/include/prereq.mk) — 前置检查
- [include/prereq-build.mk](file:///workspace/include/prereq-build.mk) — 构建前置检查
- [include/version.mk](file:///workspace/include/version.mk) — 版本定义
- [include/kernel-version.mk](file:///workspace/include/kernel-version.mk) — 内核版本解析
- [include/hardening.mk](file:///workspace/include/hardening.mk) — 加固标志
- [include/scan.mk](file:///workspace/include/scan.mk) — 元数据扫描
- [include/unpack.mk](file:///workspace/include/unpack.mk) — 源码解包

### toolchain/ 工具链

- [toolchain/Makefile](file:///workspace/toolchain/Makefile) — 工具链总调度
- [toolchain/Config.in](file:///workspace/toolchain/Config.in) — 工具链配置
- [toolchain/binutils/](file:///workspace/toolchain/binutils/) — binutils
- [toolchain/gcc/](file:///workspace/toolchain/gcc/) — GCC（minimal/initial/final）
- [toolchain/glibc/](file:///workspace/toolchain/glibc/) — glibc
- [toolchain/musl/](file:///workspace/toolchain/musl/) — musl
- [toolchain/kernel-headers/](file:///workspace/toolchain/kernel-headers/) — 内核头文件
- [toolchain/gdb/](file:///workspace/toolchain/gdb/) — GDB
- [toolchain/wrapper/](file:///workspace/toolchain/wrapper/) — 外部工具链包装器

### target/ 目标平台

- [target/Makefile](file:///workspace/target/Makefile) — 目标系统协调
- [target/Config.in](file:///workspace/target/Config.in) — 硬件特性与架构
- [target/linux/](file:///workspace/target/linux/) — Linux 内核目标支持
- [target/imagebuilder/](file:///workspace/target/imagebuilder/) — Image Builder
- [target/sdk/](file:///workspace/target/sdk/) — SDK 生成
- [target/toolchain/](file:///workspace/target/toolchain/) — 预编译工具链打包
- [target/llvm-bpf/](file:///workspace/target/llvm-bpf/) — LLVM BPF 工具链

### package/ 软件包

- [package/Makefile](file:///workspace/package/Makefile) — 主包构建协调
- [package/base-files/Makefile](file:///workspace/package/base-files/Makefile) — 基础文件系统包
- [package/base-files/image-config.in](file:///workspace/package/base-files/image-config.in) — 镜像配置

### scripts/ 脚本

- [scripts/feeds](file:///workspace/scripts/feeds) — Feed 管理
- [scripts/download.pl](file:///workspace/scripts/download.pl) — 下载脚本
- [scripts/metadata.pm](file:///workspace/scripts/metadata.pm) — 元数据模块
- [scripts/package-metadata.pl](file:///workspace/scripts/package-metadata.pl) — 包元数据处理
- [scripts/target-metadata.pl](file:///workspace/scripts/target-metadata.pl) — 目标元数据处理
- [scripts/ipkg-build](file:///workspace/scripts/ipkg-build) — IPK 包构建器
- [scripts/ext-toolchain.sh](file:///workspace/scripts/ext-toolchain.sh) — 外部工具链管理
- [scripts/getver.sh](file:///workspace/scripts/getver.sh) — 版本获取
- [scripts/diffconfig.sh](file:///workspace/scripts/diffconfig.sh) — 配置差异生成
- [scripts/config/](file:///workspace/scripts/config/) — Kconfig 配置工具

### config/ 配置

- [config/Config-build.in](file:///workspace/config/Config-build.in) — 全局构建设置
- [config/Config-devel.in](file:///workspace/config/Config-devel.in) — 开发者选项
- [config/Config-images.in](file:///workspace/config/Config-images.in) — 镜像配置
- [config/Config-ipq.in](file:///workspace/config/Config-ipq.in) — IPQ 专属选项
- [config/Config-kernel.in](file:///workspace/config/Config-kernel.in) — 内核功能配置

---

*本文档基于 ImmortalWrt AI Edition 仓库（main 分支，高通专用）生成，覆盖项目整体架构、主要模块职责、关键类与函数说明、依赖关系以及项目运行方式等关键信息。*
