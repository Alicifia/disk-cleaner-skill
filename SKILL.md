---
name: disk-cleaner
description: 安全清理磁盘垃圾文件的标准流程。当用户要求清理磁盘、释放空间、删除垃圾/缓存/临时文件时触发，适用于任意目标：整个分区（如 C 盘、D 盘）或指定文件夹。核心原则：按名字分层筛查以节省额度、只读文件片段确认身份、删除前列出清单等用户明确确认。
---

# Disk Cleaner 安全清理流程

对指定分区或文件夹做分层筛查，找出可安全删除的垃圾文件。全程遵守三条铁律：

1. **额度最小化**：先看名字判断，再决定要不要看内容。
2. **不误删**：程序文件、用户个人数据、系统核心文件一律跳过。
3. **确认后删**：任何删除前，列出具体清单（路径 + 大小 + 身份说明），等用户明确同意。

## 流程

### 第 0 步：确定目标范围并记录基线

- 确认用户要清理的目标：整个分区还是某个文件夹。未指定时询问。
- 用 `df -h`（分区）或 `du -sm`（文件夹）记录清理前占用，供最后对比。

### 第 1 步：按名字分层筛查（不进文件夹）

列出目标下一层条目**仅名字**。对每个文件夹按名字分类：

- **直接跳过（不可能有垃圾）**：程序安装目录（Program Files、Steam 等）、游戏目录、明显的用户数据目录（Desktop、Documents、Pictures、Videos）、系统核心目录。
- **候选垃圾区（继续深入）**：名字含 Temp / Cache / cache / updater / Crash / Dump / Log / 随机字符（如 `XOu5toqIA7`）/ 已知安装残留。
- **递归应用**：进入候选区后对其子文件夹重复同样判断。

### 第 2 步：测量候选大小

对候选文件夹用 `du -sm` 批量测大小（设较长 timeout）。小于几 MB 的可在清单中合并为"小项"，不为它们反复展开。

### 第 3 步：读片段确认身份（仅当名字无法判断时）

- 文本文件：Read 前 5–10 行，确认类型（日志/缓存/代码/工程文件）即停。
- 二进制或读不了：按名字 + 位置推断，归入"待确认"。
- **发现用户内容（工程文件、代码、文档）时在清单中标注 ⚠️，让用户单独决定**，不混入"安全"类别。

### 第 4 步：出清单，等确认

分三档呈现：

| 档位 | 内容 | 默认动作 |
|---|---|---|
| 建议清理 | 各类缓存、临时文件、更新器缓存、旧日志、空缓存、安装解压残留 | 等用户确认后删 |
| 需用户判断 ⚠️ | 可能是用户内容（草稿、代码、配置、注册文件） | 逐个问 |
| 不建议动 | 系统/程序必需（见下方黑名单） | 说明原因，不删 |

### 第 5 步：执行删除并验收

- 批量 `rm -rf`，对 "Device or resource busy" / 权限错误**跳过并重试无意义时不重试**，记录为"被占用，重启后释放"。
- 删完用第 0 步基线对比，报告释放空间、失败项及原因。

## 已知安全的清理目标（Windows）

- 用户级：`AppData\Local\Temp`、`*-updater`（各软件更新器缓存）、`uv` / `pip` / `npm-cache` / `pypa` / `conda`（包管理器缓存）、`CrashDumps` / `Sentry` / `drmingw`（崩溃转储）、`Downloaded Installations`、`D3DSCache`、`Temporary Internet Files`、Roaming 下的 `*_cache` / `*Log*`。
- 系统级：`Windows\Temp`、`Windows\SystemTemp`、`Windows\CbsTemp`、`Windows\Minidump`、`Windows\Panther`、`Windows\SoftwareDistribution\Download`、`Windows\Logs` 中 7 天前的旧日志（`find ... -mtime +7 -delete`）。
- 分区根目录：随机命名文件夹（安装解压残留）、散落的 `.log` / 空缓存 JSON / 明显废弃的草稿与脚本（需读片段确认后归入对应档位）。

## 黑名单（永不删除，只在清单中说明）

- `C:\Windows\Installer`（MSI 卸载缓存，删后软件无法卸载/修复）、`WinSxS`、`System32`、assembly
- `pagefile.sys`、`swapfile.sys`、`hiberfil.sys`、`DumpStack.log.tmp`、`NTUSER.DAT*`、各 `desktop.ini`、`.gitconfig`
- 用户个人文件（桌面/文档/图片/视频/下载）、程序设置目录（Roaming 下程序文件夹）、游戏存档
- 正在使用的锁定文件：不强杀进程，标注"重启后可再清"

## 输出格式

最终报告包含：释放空间（前后对比）、已删清单按类别汇总、未能删除项及原因、按约定未动的黑名单项。全程用用户的语言汇报。

---

Copyright (c) 2026 Alicifia. Released under the MIT License.
