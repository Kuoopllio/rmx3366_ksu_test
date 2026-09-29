# patches/ — 本目录下的补丁来源说明

这些补丁**不是本项目原创**，是从公开开源项目中获取并固化到本仓库的副本，
目的是让 CI 构建不依赖外部链接的可用性（原始链接一旦失效/改动，构建就会失败）。

## 文件来源

| 文件 | 来源 | 说明 |
|---|---|---|
| `susfs_patch_to_4.19.patch` | [JackA1ltman/NonGKI_Kernel_Build_2nd](https://github.com/JackA1ltman/NonGKI_Kernel_Build_2nd) `mainline` 分支 `Patches/Patch/` | SUSFS 内核补丁，适用于 Linux 4.19 内核，内含 `SUSFS_VERSION "v2.3.0"` |
| `susfs_inline_hook_patches.sh` | 同上，`Patches/` | SUSFS inline hook 注入脚本（脚本头注明 "This Hook is only available for SUSFS v2.3.00 onwards."），向 11 个内核源文件注入 `ksu_handle_*` 调用点 |
| `syscall_hook_patches.sh` | 同上，`Patches/` | 备用的 syscall hook 注入脚本（不使用 SUSFS 时使用） |
| `kona_cos15_a15__susfs_fixed.patch` | [JackA1ltman/NonGKI_Kernel_Patches](https://github.com/JackA1ltman/NonGKI_Kernel_Patches) `op_kernel` 分支 `kona_cos15_a15/` | 针对 **Kona 平台 (SM8250) + OPPO/ColorOS 内核树** 的 SUSFS 冲突修正补丁 |
| `kona_oos__susfs_fixed.patch` | 同上，`kona_oos/` | 同上，针对 Kona + OxygenOS 内核树的变体 |

## 为什么选 Kona 系列

本机 realme GT 大师探索版 RMX3366 使用 SM8250-AC（骁龙 870），属高通 **Kona** 平台；
内核源码 `ermyltsor/android_kernel_realme_sm8250` (`rui3.0`) 是 **OPPO 系**内核树。
`kona_cos15_a15` 修正补丁的上下文中出现了 `CONFIG_OPLUS_SECURE_GUARD`、
`CONFIG_OPLUS_MOUNT_BLOCK`、`CONFIG_OPLUS_KEVENT_UPLOAD` 等 OPPO 专有配置，
证明其目标内核树与本项目同族，故作为首选修正补丁。

## 许可证

上述补丁分别来自 SUSFS (simonpunk)、KernelSU 生态及 JackA1ltman 的构建项目，
遵循其原始许可证（GPL-2.0-only / GPL-3.0）。此目录仅作构建输入使用。
