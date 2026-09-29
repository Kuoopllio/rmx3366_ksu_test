# patches/ — 本目录下的补丁来源说明

这些补丁**不是本项目原创**，是从公开开源项目中获取并固化到本仓库的副本，
目的是让 CI 构建不依赖外部链接的可用性（原始链接一旦失效/改动，构建就会失败）。
唯一例外是 `kona_realme__susfs_fdinfo_fixed.patch`，它由本项目针对本机内核树生成（见下）。

## 文件来源

| 文件 | 来源 | 说明 |
|---|---|---|
| `susfs_patch_to_4.19.patch` | [JackA1ltman/NonGKI_Kernel_Build_2nd](https://github.com/JackA1ltman/NonGKI_Kernel_Build_2nd) `mainline` 分支 `Patches/Patch/` | SUSFS 内核补丁，适用 Linux 4.19，内含 `#define SUSFS_VERSION "v2.3.0"`。共 20 个文件 |
| `susfs_inline_hook_patches.sh` | 同上，`Patches/` | SUSFS inline hook 注入脚本（头注 "This Hook is only available for SUSFS v2.3.00 onwards."），向 11 个内核源文件注入 `ksu_handle_*` 调用点 |
| `syscall_hook_patches.sh` | 同上，`Patches/` | syscall hook 注入脚本（**本仓库未使用**，ReSukiSU 将 `ksu_vfs_read_hook` 列为不兼容符号） |
| `kona_oos__susfs_fixed.patch` | [JackA1ltman/NonGKI_Kernel_Patches](https://github.com/JackA1ltman/NonGKI_Kernel_Patches) `op_kernel` 分支 `kona_oos/` | 补上通用补丁在 `fs/namespace.c` 上被拒的那 1 个 hunk（SUS_MOUNT 核心定义） |
| `kona_cos15_a15__susfs_fixed.patch` | 同上，`kona_cos15_a15/` | 前者的 ColorOS 变体，`fs/namespace.c` 部分与 oos 版**逐字节等价**，但额外去动 `fs/proc/task_mmu.c` 和 `kernel/sys.c`，这两个 hunk 在本机树上**必然失败** → 因此本项目**选用 oos 版** |
| `kona_realme__susfs_fdinfo_fixed.patch` | **本项目生成** | 见下节 |

## `kona_realme__susfs_fdinfo_fixed.patch`（本项目生成，已本地严格验证）

通用补丁对 `fs/notify/fdinfo.c` 有 4 个 hunk。hunk #1~#3 在原位干净应用，
**hunk #4 被拒**。被拒原因，以及直接照搬会踩的坑：

1. hunk #4 的**尾随上下文**按更新内核写成 `inotify_mark_user_mask(mark)`，
   而本机 4.19 OPPO 树里 `inotify_fdinfo()` 仍然是
   `u32 mask = mark->mask & IN_ALL_EVENTS;` + `mask, mark->ignored_mask` → 上下文不匹配。
2. hunk #4 插入的代码**自己调用了 `inotify_mark_user_mask(mark)`**，
   而该 helper 在本机树中**不存在**（`fs/notify/inotify/inotify.h`、`fsnotify_backend.h` 都没有）
   → 如果只是把 hunk 原样重打一遍，**编译不过**。

本补丁因此：

- 锚定本机树的真实代码，把 SUS_MOUNT / SUS_KSTAT 拦截块插到
  `u32 mask = mark->mask & IN_ALL_EVENTS;` **声明之后**
  （保持 C89 声明顺序，避免 `-Wdeclaration-after-statement`）
- 把缺失的 helper 替换为本机已有的 `mask` 变量（共 2 处）
- 顺带把 `fanotify_fdinfo` 对齐成与 `show_fdinfo()` 相同的 3 参数签名
  （通用补丁把 `show_fdinfo` 改成 3 参数函数指针，却漏了 fanotify 这个调用点）

验证方式：本地取 `rui3.0` 原始 `fs/notify/fdinfo.c`，按顺序严格（fuzz 0）应用通用补丁
hunk #1~#3 复现出 CI 的真实文件状态，再严格应用本补丁，并断言：

`inotify_mark_user_mask` 已无残留 / SUS_MOUNT 块存在 / SUS_KSTAT 块存在 /
`orig_flow` 标签存在 / 原 `seq_printf` 路径完整 / fanotify 两个分支签名正确 /
插入位置在 `u32 mask` 声明之后。全部通过。

## 构建中被主动清理的死代码

`kona_oos` / `kona_cos15_a15` 两个修正补丁各自带入了

```c
static DEFINE_IDA(susfs_mnt_id_ida);
static DEFINE_IDA(susfs_mnt_group_ida);
```

这是 **SUSFS v2.2.0 时代**的遗留（v2.3.0 的通用补丁里 `susfs_mnt_id_ida`
出现 **0 次**，从未引用）。它们是 `static` 变量，编译器已明确报
`-Wunused-variable`，证明在本编译单元内无任何引用 → 工作流中直接删除，
使整个内核构建**零告警**。

## 为什么选 Kona / OPPO 系列

本机 realme GT 大师探索版 RMX3366 使用 SM8250-AC（骁龙 870），属高通 **Kona** 平台；
内核源码 `ermyltsor/android_kernel_realme_sm8250` (`rui3.0`) 是 **OPPO 系**内核树。
`kona_*` 修正补丁的上下文里出现 `CONFIG_OPLUS_SECURE_GUARD`、
`CONFIG_OPLUS_MOUNT_BLOCK`、`CONFIG_OPLUS_KEVENT_UPLOAD`，
证明其目标内核树与本项目同族，故作为首选修正补丁。

参考项目 `NonGKI_Kernel_Build_2nd` 的 OnePlus 8（SM8250）路线与本项目
**同平台、同内核版本、同 `vendor/kona-perf_defconfig`**，是最贴近的对照样板。

## 许可证

上述补丁分别来自 SUSFS (simonpunk)、KernelSU / ReSukiSU 生态及 JackA1ltman 的构建项目，
遵循其原始许可证（GPL-2.0-only / GPL-3.0）。此目录仅作构建输入使用。
