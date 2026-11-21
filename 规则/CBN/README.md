# 规则镜像（CBN 项目）

本目录是 **项目规则在 GitHub 上的镜像**，方便接手会话（Gemini 等）在一个地方读全上下文。

> **权威副本**仍在本机：`D:\deepseek-harness\规则\游戏\CBN\`（本地 git 仓库，未配远端）。
> 冲突时以本地规则目录为准；本目录只是同步副本，改这里**不会**自动影响本地规则。

## 文件

| 文件 | 原路径 | 作用 |
| --- | --- | --- |
| `AGENTS.md` | `规则\游戏\CBN\AGENTS.md` | 项目级规则：目录结构、构建命令、编码规范指针、文档/PR 分期、禁止事项、模块规则加载表 |
| `开发流程.md` | `规则\游戏\CBN\开发流程.md` | 每个 PR 的十二步固定动作与验收判据 |
| `项目状态.md` | `规则\游戏\CBN\项目状态.md` | 分支、未提交量、PR 进度、验收现状（**快照属性，可能滞后于最新提交**） |
| `模块规则\UI基础架构\AGENTS.md` | 同路径 | 改 `cata_imgui.*` / `sdltiles.cpp` 渲染与事件路径时的领域契约 |
| `工作区-README.md` | `CBN-Development\README.md` | 工作区总览：目录、构建、运行、常用命令 |
| `工作区-AGENTS指针.md` | `CBN-Development\AGENTS.md` | 工作区入口指针（3 行） |

## 注意

- 这些规则里的路径都是**本机绝对路径**（`D:\deepseek-harness\...`），换机器需要改。
- 规则只约束**本项目的开发流程与边界**，不是上游 CBN 的贡献规范；向 `cataclysmbnteam/Cataclysm-BN` 提 PR 时仍按仓库内 `AGENTS.md` / `docs/en/contribute/` 执行。
- 全局的个人安全/授权规则（`~\.dsh\AGENTS.md`）**不在**本项目规则内，也未镜像到这里。
