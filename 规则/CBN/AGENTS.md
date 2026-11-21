# Cataclysm: Bright Nights（CBN）· UI 2.0 开发

适用：谈论 CBN、改 `repo\` 代码、跑构建、写阶段文档时。基线 `a3945af`（ZIP 镜像）/ 基线提交 `0081730`。

## 1. 项目定位与目录

| 路径 | 内容 |
| --- | --- |
| `D:\deepseek-harness\game\CBN-Development\` | 项目根，入口是 `README.md` |
| `repo\` | 主工程（git），分支 `feature/cbn-ui-2.0`，工作目录起点 |
| `docs\` | 阶段审计/原型/排查文档 + 截图 |
| `archive\` | 上游源码快照 ZIP + `src-mirror\` 解包只读镜像 |
| `local\` | 本地构建覆盖配置（**不进 git**） |
| `vcpkg\` | 目录联接 → `D:\vcpkg`（跨项目共用工具链，不随项目走） |
| `build.log` / `rebuild.ps1` | 最近构建日志 / 一键重建脚本 |

安装版 `C:\Users\Admin\Desktop\cbn-windows-tiles-x64-msvc-2026-11-08` 是独立副本（基线 `5e8f57d`），**不是**开发目标。

## 2. 构建与运行

构建**必须**带两个参数，缺一不可（原因：vcpkg 是 ZIP 快照无 git 历史，manifest 基线解析不了；上游 `build-scripts/MSVC.cmake` 强制 `/Z7`，Release 下 obj 约 4.9 GB，聚合静态库触发 LNK1248）：

```powershell
$vs = 'C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools'
$env:DevEnvDir = "$vs\Common7\IDE\"
$env:PATH = "$vs\Common7\IDE\CommonExtensions\Microsoft\CMake\CMake\bin;" +
            "$vs\Common7\IDE\CommonExtensions\Microsoft\CMake\Ninja;" + $env:PATH
$cmake = "$vs\Common7\IDE\CommonExtensions\Microsoft\CMake\CMake\bin\cmake.exe"

cd D:\deepseek-harness\game\CBN-Development\repo
& $cmake --preset windows-tiles-sounds-x64-msvc `
    -DVCPKG_MANIFEST_MODE=OFF `
    -DCMAKE_PROJECT_INCLUDE=D:\deepseek-harness\game\CBN-Development\local\cbn-local-override.cmake
& $cmake --build out/build/windows-tiles-sounds-x64-msvc --config Release
```

- 一键：`powershell -ExecutionPolicy Bypass -File D:\deepseek-harness\game\CBN-Development\rebuild.ps1`
- 产物：`repo\out\build\windows-tiles-sounds-x64-msvc\src\Release\cataclysm-bn-tiles.exe`、`cata_test-tiles.exe`
- `local\cbn-local-override.cmake` 只做一件事：`CMAKE_MSVC_DEBUG_INFORMATION_FORMAT=ProgramDatabase`

运行：

```powershell
cd D:\deepseek-harness\game\CBN-Development\repo   # 工作目录必须是源码根
$env:CBN_IMGUI = '1'
$env:CBN_IMGUI_FONT = 'D:\deepseek-harness\game\CBN-Development\repo\font\ZhenKai_GBK-Regular.ttf'
.\out\build\windows-tiles-sounds-x64-msvc\src\Release\cataclysm-bn-tiles.exe
```

| 入口 | 作用 |
| --- | --- |
| `CBN_IMGUI=1` | 开发期快捷开关，置位即初始化并自动打开原型窗口（无需存档） |
| `CBN_IMGUI_FONT=<路径>` | 指定 CJK 字体；未设时依次回退 `user_fontdir` → `fontdir` 的 `ZhenKai_GBK-Regular.ttf` → `unifont.ttf` → `Terminus.ttf` |
| 选项 `IMGUI_PROTOTYPE` | Settings → Graphics，默认关，改动需重启 |
| `F10` | 运行时切换原型窗口显示 |

关闭上述开关时 `init()` 直接返回，零开销、零行为变化；所有新代码在 `#if defined(TILES)` 内，curses 构建不受影响。

## 3. 编码规范

**不要在本文件重复**。C++ 风格、Lua 注解、提交信息、格式化与测试命令，一律以 `repo\AGENTS.md` 为准（工作目录在 `repo\` 下时会自动加载）。本文只补项目级约束。

## 4. 文档与 PR 分期约定

- 阶段文档落 `docs\`，命名 `YYYY-MM-DD-CBN-<阶段>-<主题>.md`（实例：`2026-11-20-CBN-UI2.0-PR0-审计.md`、`2026-11-20-CBN-UI2.0-PR1-ImGui原型.md`）。
- 每份必须含：目标 / 改动文件清单 / 实测验证（命令 + 时间 + 产物）/ 验收项对照表 / 未完成与待确认。
- **未测量不下结论**：性能、回归、全量测试没跑就写「未测量」，不写「无影响」。
- **验收停点**：阶段结束时明确写「停在 PR-N，等待验收，不进入 PR-N+1」，并在会话里复述。未获用户验收不推进下一阶段。
- 当前进度：PR-0 / 0.5 / 0.6 / PR-1 已完成，**PR-1 等待验收**（交互实测、shutdown 接入均未做）。进度快照以 `项目状态.md` 为准。
- 流程按 `开发流程.md` 的十二步走；**代码已实现 / 编译成功 / 测试通过 / 正式验收是四种状态**，不许混用。
- 大段输出按全局约定落 `_out\`，会话里只回摘要。

## 5. 禁止事项

- 不进 `archive\src-mirror\` 改文件（只读对照，内容等同基线提交）。
- 不覆盖、不改动安装版目录。
- 不把 `local\` 内容提交进 git；不擅自 push、改远端、`git clean`/`reset --hard`。
- 不启用 `repo\CMakeUserPresets.json`（空 `name` 会让 CMake 3.31 读不了 preset 列表，已改名 `.disabled`，备份 `.bak-20261120`）。
- 不删除 `out\build` 除非明确要求（重建约 8.5 分钟，迁移后必须先 configure 才能用）。
- 未获验收不自行进入下一 PR，不扩大改动到制作台/AIM/物品/NPC/Zone 逻辑。
- 不引入未说明的新依赖（当前仅 vcpkg `imgui[core,sdl3-binding,sdl3-renderer-binding]:x64-windows-static` 1.93.0）。

## 6. 维护边界

- 只在用户明确提到「更新 README / 改规则 / 迁移目录」时才动本文件与 `README.md`；平时不主动整理、不主动 commit。
- 已知失败用例清单等项目本地事实放 `local\`，不进 git。
- 结构变化（迁移、改名、换基线、换构建参数）后：同步 `README.md` 与本文件 → 在 `D:\deepseek-harness\规则` 跑 `node tools\rules-check.mjs --fix` 刷新工作区指针哈希。
- 安全与授权规则一律不进本文件，只放 `~\.dsh\AGENTS.md`。

## 7. 按需加载（模块规则）

主文件只放项目级约定，**模块级技术约束按需加载**：

| 路径 | 何时读 |
| --- | --- |
| `模块规则\UI基础架构\AGENTS.md` | 改 `cata_imgui.*`、`sdltiles.cpp` 渲染/事件路径、字体/IME/开关；含交互领域契约、键盘捕获约束、阶段边界 |
| `项目状态.md` | 开工前确认当前分支、未提交量、待办 |
| `开发流程.md` | 每个 PR 的固定动作与验收判据 |

- 交互标准与阶段路线（Hybrid Interaction、PR-1→PR-1.5→PR-2→…→PR-7）以 `game\CBN-Development\docs\2026-11-21-CBN-UI2.0-混合式交互标准与路线图.md` 为准；本文件只登记入口，不复制路线表。
- **PR-1 与 PR-1.5 分别验收**：PR-1 只做输入底层（含键盘分流），PR-1.5 才做窗口/标签页/右键菜单/列表/焦点导航。不得合并成一个无限扩大的 PR-1。

- 未写模块规则前 **不得** 凭空引用模块约束；已规划未建立：AIM、装备、容器、制作、Zone —— 等该模块进入设计阶段再写，不在本次预造。
- 模块规则只写该模块的技术事实与红线，不重复本文件内容；新增模块规则后在此表登记。
- 会话中确认要动某模块时，先读一次对应文件，不要靠记忆。
