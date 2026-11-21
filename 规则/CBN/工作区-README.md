# CBN-Development 管理入口

Cataclysm: Bright Nights（CBN）UI 2.0 项目的统一入口。所有和 CBN 有关的东西从这里找。

建立于 2026-11-20，同日完成物理迁移（原 `D:\CBN-Development` 已并入本目录）。

## 目录一览

```
D:\deepseek-harness\game\CBN-Development\
├── repo\      主工程（git 仓库，10+ GB，含构建产物）
├── docs\      审计 / 原型 / 排查文档 7 份 + 原型截图
├── archive\   
│   ├── cbn-src-a3945af.zip     上游源码快照（97.7 MB）
│   └── src-mirror\             解包后的只读源码镜像（175 MB，审计对照用）
├── local\     本地构建覆盖配置（不进 git）
├── vcpkg\  →  D:\vcpkg（目录联接，依赖管理器 3.2 GB）
├── README.md  本文件
├── rebuild.ps1  迁移后的重建脚本（configure + build）
└── build.log  最近一次构建日志
```

`repo`、`docs`、`archive`、`local` 都是真实文件，已经不依赖任何外部路径。
只有 `vcpkg` 仍是联接：它是跨项目共用的工具链（3.2 GB），不随项目走，
删联接不影响依赖本身。

## 主工程状态

```
位置    D:\deepseek-harness\game\CBN-Development\repo
分支    feature/cbn-ui-2.0
基线    0081730  baseline: Cataclysm-BN a3945af (ZIP mirror, unmodified)
未提交  M src/CMakeLists.txt
        M src/options.cpp
        M src/sdltiles.cpp
        ?? src/cata_imgui.cpp
        ?? src/cata_imgui.h
产物    out\build\windows-tiles-sounds-x64-msvc\src\Release\cataclysm-bn-tiles.exe
```

## 构建

迁移后构建缓存（`out\build`，8.7 GB）里写死了旧绝对路径，已删除重建。

**注意**：`CMakeUserPresets.json` 是 VS 生成的，`name` 为空会让 CMake 3.31
读不了 preset 列表，已改名为 `CMakeUserPresets.json.disabled`
（备份 `CMakeUserPresets.json.bak-20261120`）。

```powershell
$vs = 'C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools'
$env:DevEnvDir = "$vs\Common7\IDE\"
$env:PATH = "$vs\Common7\IDE\CommonExtensions\Microsoft\CMake\CMake\bin;" +
            "$vs\Common7\IDE\CommonExtensions\Microsoft\CMake\Ninja;" + $env:PATH
$cmake = "$vs\Common7\IDE\CommonExtensions\Microsoft\CMake\CMake\bin\cmake.exe"

cd D:\deepseek-harness\game\CBN-Development\repo

# 配置（两个参数缺一不可）
& $cmake --preset windows-tiles-sounds-x64-msvc `
    -DVCPKG_MANIFEST_MODE=OFF `
    -DCMAKE_PROJECT_INCLUDE=D:\deepseek-harness\game\CBN-Development\local\cbn-local-override.cmake

# 构建
& $cmake --build out/build/windows-tiles-sounds-x64-msvc --config Release
```

一键重建：`powershell -ExecutionPolicy Bypass -File .\rebuild.ps1`

### 迁移后重建结果（2026-11-20 21:45）

| 项 | 结果 |
| --- | --- |
| configure | 成功，8.8s |
| Release 编译 | 成功，8.5 分钟 |
| obj | 681 个 |
| 错误 | 0（无 error C / LNK） |
| 主程序 | `cataclysm-bn-tiles.exe` 75.0 MB，21:45:02 |
| 测试程序 | `cata_test-tiles.exe` 84.6 MB，21:44:46 |
| ImGui 原型 | `cata_imgui.cpp` 已参与编译 |

日志：`build.log`

参数原因：

- `VCPKG_MANIFEST_MODE=OFF`：vcpkg 是 ZIP 快照（无 git 历史），manifest 基线解析不了，走 classic 模式
- `CMAKE_PROJECT_INCLUDE`：把 `/Z7` 覆盖成 `ProgramDatabase`，否则 4.9 GB obj 聚合静态库会触发 LNK1248

## 运行

```powershell
cd D:\deepseek-harness\game\CBN-Development\repo    # 工作目录必须是源码根
$env:CBN_IMGUI = '1'
$env:CBN_IMGUI_FONT = 'D:\deepseek-harness\game\CBN-Development\repo\font\ZhenKai_GBK-Regular.ttf'
.\out\build\windows-tiles-sounds-x64-msvc\src\Release\cataclysm-bn-tiles.exe
```

游戏内开关：选项 → 图形 → `IMGUI_PROTOTYPE`（默认关，改完重启）。

## 文档索引

| 文档 | 内容 |
| --- | --- |
| `docs\2026-11-20-CBN-UI2.0-PR0-审计.md` | 远程仓库/提交基线核对（Go/No-Go） |
| `docs\2026-11-20-CBN-UI2.0-PR0-本地审计.md` | 本地源码审计，运行版本与提交一致性 |
| `docs\2026-11-20-CBN-UI2.0-PR0.5-环境与编译验证.md` | 编译环境、依赖、可运行性验证 |
| `docs\2026-11-20-CBN-UI2.0-PR0.6-基线收尾.md` | 本地覆盖配置落地、目录隔离约定 |
| `docs\2026-11-20-CBN-UI2.0-PR1-ImGui原型.md` | PR-1 设计与改动清单 |
| `docs\2026-11-10-CBN模组与本体排查.md` | 早期模组/本体排查记录 |
| `docs\2026-11-10-CBN报错处理.md` | 早期报错处理记录 |

以上文档里的绝对路径已同步更新为本目录下的新路径。

## 约定

- **源码镜像只读**：`archive\src-mirror\Cataclysm-BN-a3945af…` 是上游同提交的解包副本，
  内容与 `repo\` 的基线提交一致，**不要在镜像里改文件**，审计/比对时用它做只读对照。
- **本地配置隔离**：构建脚本、已知失败清单放 `local\`，不提交进仓库。
- **不覆盖安装版**：游戏安装目录 `C:\Users\Admin\Desktop\cbn-windows-tiles-x64-msvc-2026-11-08`
  是独立副本（git 基线 `5e8f57d`），开发只改 `repo\`。

## 变更记录

- 2026-11-20
  - 建立本目录，整理 7 份文档 + 原型截图到 `docs\`，`cbn-src.zip` 归档到 `archive\`
  - 物理迁移：`D:\CBN-Development\CBN-UI2` → `repo\`，`_local` → `local\`，原目录清空
  - 删除失效的 `out\build`（8.7 GB），重新 configure
  - `CMakeUserPresets.json` 因空 name 禁用（改名 `.disabled`，保留 `.bak-20261120`）
  - 文档内绝对路径批量更新
  - 重新构建通过：主程序 75 MB、测试程序 84.6 MB，0 错误
  - 源码镜像 `_work\src` 迁入 `archive\src-mirror\`，文档引用同步更新
