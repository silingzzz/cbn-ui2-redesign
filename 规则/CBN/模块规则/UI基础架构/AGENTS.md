# CBN 模块规则 · UI 基础架构

按需加载：**只有做 UI 层改动时才读本文件**，由 `..\..\AGENTS.md` §7 指路。
事实来源：`repo\src\cata_imgui.cpp`（211 行）、`repo\src\cata_imgui.h`（35 行）、`repo\src\sdltiles.cpp` 三处钩子、`docs\2026-11-20-CBN-UI2.0-PR1-ImGui原型.md`。

## 1. 现行架构（PR-1 实测，非设计愿景）

```
WinCreate()                                CheckMessages()
  创建 SDL_Window + SDL_Renderer             while( SDL_PollEvent )
  → cata_imgui::init(win, renderer)            → cata_imgui::process_event(ev)
                                              真 = 游戏跳过该事件
refresh_display()
  RenderCopy(display_buffer → 窗口)
  → cata_imgui::render(renderer)
  → SDL_RenderPresent
```

- 后端：`imgui_impl_sdl3`（平台/输入）+ `imgui_impl_sdlrenderer3`（渲染），与游戏共用同一个 `SDL_Renderer`，**不新建窗口、不新建渲染目标**。
- 接口仅 7 个函数（`cata_imgui.h`）：`initialized / render / process_event / init / shutdown / is_open / toggle`。
- 实现细节（`cata_imgui.cpp` 匿名命名空间）：`g_initialized`、`g_open`、`g_window`、`g_renderer` 四个状态；原型窗口含帧率、`InputText`、`Checkbox`、`SliderInt`、`关闭` 按钮。

## 2. 生命周期约束（硬性）

1. `init()` **必须**在 `SDL_CreateRenderer` 成功之后调用（现位于 `sdltiles.cpp` WinCreate 内、全屏分支之前）。过早调用拿不到 renderer，会静默不初始化。
2. `init()` 自带幂等与空指针保护，**不要**在调用点再判一次。
3. `shutdown()` 必须早于 `renderer.reset()` / `::window.reset()`（`sdltiles.cpp` 约 L422–424 的清理路径）。后端持有 window/renderer 原始指针，晚于销毁会野指针。
4. **当前 `shutdown()` 未接入任何退出路径**（PR-1 §9.3 遗留）。任何新增后端资源、新窗口或退出期回调前，**必须先把 shutdown 接入清理流程**，否则退出泄漏与销毁顺序问题会在新代码上放大。
5. 关闭顺序固定：`ImGui_ImplSDLRenderer3_Shutdown()` → `ImGui_ImplSDL3_Shutdown()` → `ImGui::DestroyContext()`。

## 3. 每帧顺序（硬性）

```
ImGui_ImplSDLRenderer3_NewFrame()
ImGui_ImplSDL3_NewFrame()
ImGui::NewFrame()
...绘制...
ImGui::Render()
ImGui_ImplSDLRenderer3_RenderDrawData( draw_data, renderer )
SDL_RenderPresent( renderer )
```

- 两个后端 `NewFrame()` **必须**在 `ImGui::NewFrame()` 之前，颠倒会触发 ImGui 断言。
- `render()` 内有 `renderer != g_renderer` 守卫；不要移除（窗口重建时 renderer 会换）。
- ImGui 画在主帧 buffer 之上、`SDL_RenderPresent` 之前；**不要**改 `RenderCopy` 与 present 的相对顺序。
- 禁止在 `render()` 里创建/销毁纹理以外的重量级资源；每帧只应做绘制。

## 4. 事件分发与输入隔离

- 事件唯一入口是 `process_event()`，由 `CheckMessages()` 的 `SDL_PollEvent` 循环“喂进去、返回真则 `continue`”。**不得**在别处另开事件分支。
- 返回 `true` 的判据（现行实现，按事件类型白名单）：
  - 鼠标（motion / button up/down / wheel）→ `io.WantCaptureMouse`
  - 键盘与文本（key down / key up / text input / text editing）→ `io.WantCaptureKeyboard || io.WantTextInput`
  - 其余类型 → `false`（交回游戏）
- `F10` 是**保留键**：`KEY_DOWN` 且非 repeat 时切窗口并吞事件。新增快捷键必须先避让 `F10` 与现有游戏键位，且在窗口打开时才消费。
- 窗口关闭（`g_open=false`）后，`F10` 仍生效（用于重新打开），但其余事件不再吞。

## 5. 中文输入法与字体

- IME：`toggle()` 在打开时 `SDL_StartTextInput(g_window)`、关闭时 `SDL_StopTextInput()`；`SDL_EVENT_TEXT_INPUT` / `SDL_EVENT_TEXT_EDITING` 已转发给 ImGui。
- 字体加载顺序（`load_cjk_font()`，仅此一处）：`CBN_IMGUI_FONT` → `user_fontdir` → `fontdir` 的 `ZhenKai_GBK-Regular.ttf` → `fontdir\unifont.ttf` → `fontdir\Terminus.ttf`，字号 18.0f，字形范围 `GetGlyphRangesChineseSimplifiedCommon()`。
- **字体图集必须在 `ImGui_ImplSDL3_InitForSDLRenderer()` 之前构建**（后端初始化时会烤纹理）。在 init 之后 `AddFontFromFileTTF` 不生效——这是实测踩过的坑。
- 新增字体：只往 candidates 链里加，不要另写一套加载逻辑；不允许在 init 之后加字体。

## 6. 资源与开关

- `io.IniFilename = nullptr` **必须保持**：否则 ImGui 会在 cwd（= 源码根）写 `imgui.ini`，污染仓库。
- 开关双入口：选项 `IMGUI_PROTOTYPE`（graphics 页，默认 `false`，`COPT_CURSES_HIDE`，需重启）+ 环境变量 `CBN_IMGUI`（置位即初始化并自动开窗，开发期用）。
- 两者皆关时 `init()` 直接 return —— **零开销、零行为变化**，这是验收基线，不许破坏。
- 事件循环里不得新增每帧无条件堆分配；原型窗口本身的开销未做基准，不得声称“无开销”。

## 7. curses 与 Tiles 共存

- 所有声明与实现都在 `#if defined( TILES )` 内；`cata_imgui.h` 只用 `struct SDL_Window;` 前置声明，不 include SDL 头。
- `IMGUI_PROTOTYPE` 选项同样包在 `#if defined( TILES )` 内，curses 构建看不到该选项。
- 新增文件加入 `src/CMakeLists.txt` 的 `if (TILES)` 块，**不许**加进公共源列表。

## 8. 职责边界（UI 与业务）

- `cata_imgui.cpp` 只负责“上下文 + 后端 + 字体 + 事件 + 绘制”。**不得**把游戏状态写入、存档、物品/怪物逻辑放进这里。
- UI 读游戏数据走既有只读接口，写操作必须回到游戏原有的 action/交互路径，不得在 UI 层直接改 `avatar`/`Character` 状态。
- 原型窗口里的 `g_text_buffer` 等是**占位控件**，不是产品 UI，不要在其上堆业务。

## 9. 性能与 DPI

- 已实测：主菜单 8.5 FPS，与该版本空闲帧率一致；**未做基准对比，不得引用为“已优化”**。
- 已知限制：`SDL_SetRenderLogicalPresentation`（`sdltiles.cpp` L1419 / L1620）下 ImGui 按逻辑分辨率绘制，窗口缩放时覆盖层跟随缩放；**未做 DPI 适配**，窗口尺寸按逻辑像素。PR-2 若要做真实 UI，先解决这一点。

## 10. 未完成项（跟踪，不许在其他 PR 里顺手改）

| 项 | 状态 |
| --- | --- |
| `shutdown()` 接入退出流程 | 未做 |
| F10 切换 / 鼠标点击 / IME 组字的真实交互实测 | 未做（需人工或事件注入） |
| 与上游 CDDA `cata_imgui` 的代码级对照 | 未做；**本机无 CDDA 源码，不得把 CDDA 实现细节当既成事实引用** |
| 全量测试回归 | 见 `..\..\..\..\项目状态.md` |
| DPI / 逻辑分辨率适配 | 未做 |

## 11. 交互领域与分流契约（2026-11-21 决策）

正式方向：**Hybrid Interaction** —— 新 UI 走现代鼠标交互，游戏世界保留原操作。标准与路线见 `game\CBN-Development\docs\2026-11-21-CBN-UI2.0-混合式交互标准与路线图.md`，本文只记技术契约。

- 分流唯一入口仍是 `process_event()`（ImGui 侧）与 `CheckMessages()` 的 switch（游戏侧），**不得另开第二条通路**。
- 鼠标 DOWN/UP 归属用三态 `MOUSE_OWNER_NONE / IMGUI / GAME`：按下记归属，释放查表决定去向，`WINDOW_FOCUS_LOST` 清表。规则为「按下谁收，释放归谁」，避免 UI 内按下、窗口外松开时漏给游戏或产生幽灵点击。
- 游戏侧鼠标语义不变：`MOUSE_LEFT → SELECT`、`MOUSE_RIGHT → SEC_SELECT`、`SCROLLWHEEL_*` 仅在列表类 UI 生效。**不得在 PR-1 扩展原游戏鼠标功能**。
- 新增鼠标操作必须调用原游戏业务函数，禁止在 ImGui 层重新实现游戏规则。

## 12. 键盘捕获约束（PR-1 待实施，非组件焦点系统）

现状：`want_capture_keyboard()` / `want_capture_text_input` 只提供查询，**事件循环里未接入**——键盘分流尚未实施。

实施时必须满足：

1. **KEYDOWN / KEYUP 归属一致**：用与鼠标同一套归属思路，按下时记录是否 UI 捕获，释放按记录处理；不能用「释放时再查一次 `WantCaptureKeyboard`」代替（状态可能在按住期间变化）。
2. **`F10` 例外**：F10 是覆盖层全局开关，必须在任何捕获判定之前处理并消费，保证窗口关闭时也能打开。
3. **文本输入**：`SDL_EVENT_TEXT_INPUT` / `SDL_EVENT_TEXT_EDITING` 必须转发给 ImGui；焦点在文本框时用 `io.WantTextInput` 判定，而不是 `WantCaptureKeyboard`。
4. **不捕获时行为不变**：`want_capture_keyboard()` 为假时，原游戏键盘路径逐字不变，不得顺手重构。
5. **失焦与关闭**：`WINDOW_FOCUS_LOST` 时清空键盘归属并确保游戏侧不残留按住状态（否则角色会一直前进）；覆盖层关闭时同样清理。
6. **禁止写法**：单加一行 `if( want_capture_keyboard() ) break;` 就宣布完成——那会把 `F10` 一起吞掉，且按下/释放会错配。

不属于本项：`Tab`/`Shift+Tab` 焦点导航、控件焦点链表、快捷键系统 —— 属于 PR-1.5 的组件能力。

## 13. 演示件边界

`draw_prototype_window()` 里的 `BIG TEST BUTTON`、复选框、滑块、文本框、帧率显示是**开发诊断工具**：

- 仅验证输入通道与控件响应，不算正式游戏功能；
- 不得用作 AIM 或任何业务模块已实现的证据；
- PR-1.5 可用组件测试数据；**PR-2 起必须用真实游戏数据与真实业务接口验收**；
- 正式 UI 组件另起文件，不要继续堆进 `cata_imgui.cpp` 的匿名命名空间。

## 14. 阶段边界（防无限扩大）

| 阶段 | 交付 | 明确不做 |
| --- | --- | --- |
| PR-1 基础输入框架 | 生命周期、事件接入、帧驱动、鼠标分流、键盘分流、DOWN/UP 归属、坐标与缩放、IME 通道、诊断窗口 | 窗口/标签页/右键菜单/列表/焦点导航 |
| PR-1.5 交互组件 | 窗口移动缩放、标签页、上下文菜单、滚动列表与表格、焦点管理、Tab 导航、文本框与 IME 交互、操作反馈 | 任何真实业务数据与业务操作 |
| PR-2 起 | 各业务模块，调用原游戏接口 | 用演示数据验收 |

PR-1 与 PR-1.5 **分别验收**，不得合并成一个持续扩大的 PR-1。
