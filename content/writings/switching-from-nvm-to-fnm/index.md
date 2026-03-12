---
title: "从 NVM 迁移到 FNM：解决 Zsh 启动慢问题"
date: 2026-01-27T10:06:39+08:00
description: "通过从 NVM 迁移到 FNM 解决 Zsh 终端启动慢的问题，实测性能提升 68%，从 3.1 秒降至 1.0 秒。"
categories:
- 技术实践
- 性能优化
tags:
- nodejs
- 开发工具
- 性能优化
---

最近在使用终端时遇到一个明显卡顿：每次打开新的终端窗口，Zsh 都要等待几秒才能完成加载。排查后发现罪魁祸首是 Node Version Manager (NVM) 的初始化脚本。本文记录从 NVM 迁移到 Fast Node Manager (FNM) 的过程，以及性能提升的实测结果。

<!--more-->

## 问题：NVM 拖慢 Zsh 启动速度

NVM 是 Node.js 版本管理的常用工具，但它有一个明显的缺点：启动速度慢。每次打开新终端时，`.zshrc` 中的 NVM 初始化代码都需要执行：

```bash
export NVM_DIR="$HOME/.nvm"
[ -s "/opt/homebrew/opt/nvm/nvm.sh" ] && \. "/opt/homebrew/opt/nvm/nvm.sh"
[ -s "/opt/homebrew/opt/nvm/etc/bash_completion.d/nvm" ] && \. "/opt/homebrew/opt/nvm/etc/bash_completion.d/nvm"
```

这段代码看似简单，但 `nvm.sh` 是一个大型的 Shell 脚本，加载它会显著增加终端启动时间，尤其是在频繁打开新终端窗口的情况下，这种延迟会严重影响工作效率。

### 实测启动耗时

通过 `time zsh -i -c exit` 命令测量，使用 NVM 时的终端启动时间：

```bash
# 使用 NVM 时的启动耗时
zsh -i -c exit  0.56s user 0.52s system 34% cpu 3.142 total
```

**结果：启动耗时约 3.1 秒**，其中大部分时间消耗在加载 `nvm.sh` 脚本上。这意味着每次打开终端都要额外等待超过 3 秒。

### NVM 启动慢的原因

1. **纯 Shell 脚本实现**：NVM 完全使用 Shell 脚本编写，缺乏编译型语言的性能优势
2. **复杂的初始化逻辑**：需要扫描安装的 Node 版本、设置环境变量、加载 bash 补全等
3. **每次启动都重复执行**：无论是否需要使用 Node，初始化代码都会执行

## 解决方案：迁移到 FNM

### 什么是 FNM？

Fast Node Manager (FNM) 是一个用 Rust 编写的 Node.js 版本管理器，它的设计目标就是快速和简单。与 NVM 相比，FNM 具有以下优势：

- **极快的启动速度**：Rust 编写的二进制程序，初始化只需几毫秒
- **跨平台支持**：支持 macOS、Linux 和 Windows
- **自动版本切换**：支持根据项目目录自动切换 Node 版本
- **兼容 `.nvmrc`**：无缝支持 NVM 的版本配置文件
- **简单的 API**：命令行接口清晰直观

### 迁移步骤

#### 1. 安装 FNM

如果还没安装，可以通过 Homebrew 安装：

```bash
brew install fnm
```

#### 2. 修改 `.zshrc` 配置

移除 NVM 的初始化代码，替换为 FNM 的简洁配置：

```bash
# fnm (Fast Node Manager)
eval "$(fnm env --use-on-cd)"
```

`--use-on-cd` 参数让 FNM 在切换目录时自动根据 `.nvmrc` 或 `.node-version` 文件切换 Node 版本。

#### 3. 清理旧的 NVM 安装

删除 NVM 及其安装的所有 Node 版本：

```bash
# 删除 NVM 目录
rm -rf ~/.nvm

# 如果通过 Homebrew 安装，卸载 NVM
brew uninstall nvm
```

#### 4. 安装 Node.js 版本

使用 FNM 安装需要的 Node 版本：

```bash
# 安装最新 LTS 版本
fnm install --lts

# 设置为默认版本
fnm default 24

# 查看已安装的版本
fnm list
```

## FNM 常用命令

### 版本管理

```bash
# 安装特定版本
fnm install 20.18.3
fnm install 18

# 安装最新版本
fnm install --latest

# 安装最新 LTS 版本
fnm install --lts

# 列出所有已安装的版本
fnm list

# 列出所有可用的远程版本
fnm list-remote
```

### 版本切换

```bash
# 切换到指定版本
fnm use 20

# 设置默认版本
fnm default 24

# 查看当前使用的版本
fnm current
```

### 版本卸载

```bash
# 卸载指定版本
fnm uninstall 18.16.0
```

### 项目级版本管理

FNM 支持在项目目录中使用 `.nvmrc` 或 `.node-version` 文件来指定 Node 版本：

```bash
# 在项目根目录创建 .node-version 文件
echo "20.18.3" > .node-version

# 或创建 .nvmrc 文件（兼容 NVM）
echo "20.18.3" > .nvmrc
```

当配置了 `fnm env --use-on-cd` 后，进入项目目录时会自动切换到指定的 Node 版本。如果该版本未安装，FNM 会提示你安装。

### 环境信息

```bash
# 查看 FNM 环境信息
fnm env

# 查看 FNM 版本
fnm --version
```

## 迁移后的效果

### 性能提升

从 NVM 迁移到 FNM 后，终端启动速度有了显著提升。再次使用相同的命令测量：

```bash
# 使用 FNM 后的启动耗时
zsh -i -c exit  0.25s user 0.34s system 59% cpu 0.988 total
```

**结果：启动耗时约 1.0 秒**，相比 NVM 的 3.1 秒，性能提升了 **68%**。

#### 性能对比

| 指标 | NVM | FNM | 提升幅度 |
|------|-----|-----|----------|
| 启动耗时 | 3.142 秒 | 0.988 秒 | **68% ↓** |
| User Time | 0.56s | 0.25s | 55% ↓ |
| System Time | 0.52s | 0.34s | 35% ↓ |

具体改善：
- **启动时间缩短**：从 3.1 秒降至 1.0 秒，节省 2+ 秒
- **资源占用更低**：CPU 用户态时间减少 55%
- **交互响应更快**：版本切换等操作几乎瞬间完成

### 使用体验改善

1. **更快的工作流**：频繁开关终端窗口不再有心理负担
2. **自动版本切换**：进入项目目录自动使用正确的 Node 版本，无需手动操作
3. **简洁的命令**：FNM 的命令设计更直观，学习成本低
4. **完全兼容**：支持 `.nvmrc` 文件，与 NVM 项目无缝对接

### 注意事项

1. **全局包需要重新安装**：迁移后，之前 NVM 安装的全局 npm 包需要重新安装
2. **Shell 配置更新**：记得更新 `.zshrc` 或 `.bashrc` 配置文件
3. **团队协作**：如果团队其他成员使用 NVM，`.nvmrc` 文件仍然有效

## 总结

FNM 作为 NVM 的现代替代方案，在保持功能完整性的同时，大幅提升了性能和用户体验。从 NVM 迁移到 FNM 的过程简单快捷，只需几分钟就能完成。如果你也在为终端启动速度慢而困扰，不妨试试 FNM。

迁移后的主要收获：
- ✅ 终端启动速度明显加快，日常使用更流畅
- ✅ 自动版本切换让项目间切换更顺畅
- ✅ 命令响应几乎瞬时完成
- ✅ 向后兼容 `.nvmrc` 配置文件

工具的选择应该服务于效率，而不是成为负担。FNM 在性能与易用性上对大多数开发者都更友好。

## 适用边界

- 如果团队必须统一 NVM，个人迁移需要评估协作成本。
- 某些脚本强依赖 `nvm` 命令时，需要做兼容替换。

## 参考资源

- [FNM GitHub](https://github.com/Schniz/fnm)
- [FNM 文档](https://github.com/Schniz/fnm#readme)
