# CDDA ImGui 鼠标交互对照报告

2026-11-21 · 研究对象：`CleverRaven/Cataclysm-DDA` master（commit `e422f766…`，2026-11-20）
对照对象：CBN `feature/cbn-ui-2.0`（基线 `a3945af`）+ PR-1 原型
方法：拉取 CDDA `src/cata_imgui.cpp`（49,006 B）/ `cata_imgui.h`（10,635 B）/ `src/sdltiles.cpp`（312,752 B）/ `doc/USER_INTERFACE_AND_ACCESSIBILITY.md`（9,933 B）与 CBN 本地源码逐段对照。原始文件存放在 `game\_work\cdda-ref\`。

## 0. 版本关系（先说清楚，避免误移植）

- CBN 是 CDDA 的 fork（`repo\README.md:30` 自述），但本地仓库只有 1 个基线提交，**无法从本机确定对应的 CDDA 提交**。
- 量化差异：CBN `sdltiles.cpp` **4,311 行**，CDDA master **7,587 行**；CBN 无 `cataimgui::` 命名空间、无 `CATA_*` 事件宏抽象、无 `cata::options::mouse`、无 `ui_adaptor` 驱动的 ImGui 帧。
- 结论：**CDDA master 的实现只能借鉴架构与算法，不能 cherry-pick**。需要按 CBN 现有 SDL3 原生事件类型重写接入层。

## 1. SDL 鼠标事件接入（CDDA 实际路径）

```cpp
// sdltiles.cpp:6136-6142  CheckMessages 主循环
while( SDL_PollEvent( &ev ) ) {
    SDL_Event ev_display = ev;
    convert_event_to_display_buffer_coords( &ev_display );   // 窗口像素 → display_buffer 坐标
    imclient->process_input( &ev_display, imgui_buf_w, imgui_buf_h, scaling_factor );
    ...
```

```cpp
// cata_imgui.cpp:649-685
void cataimgui::client::process_input( void *input, int, int, int scaling_factor ) {
    if( any_window_shown() ) {                    // ① 没有任何 ImGui 窗口就不喂
        const SDL_Event *evt = static_cast<const SDL_Event *>( input );
        if( !evt ) return;
        bool no_mouse = ImGui::GetIO().ConfigFlags & ImGuiConfigFlags_NoMouse;
        if( no_mouse ) {                          // ② 玩家关了鼠标，鼠标事件全部丢弃
            switch( evt->type ) { case CATA_MOUSEMOTION: case CATA_MOUSEWHEEL:
                case CATA_MOUSEBUTTONDOWN: case CATA_MOUSEBUTTONUP: return; }
        }
        SDL_Event imgui_ev = *evt;                // ③ 拷贝，不改游戏看到的那份
        if( scaling_factor > 1 ) {                // ④ 再除一次缩放（UI 缩放）
            if( imgui_ev.type == CATA_MOUSEMOTION ) { imgui_ev.motion.x /= scaling_factor; ... }
            else if( BUTTONDOWN/UP ) { imgui_ev.button.x /= scaling_factor; ... }
            else if( CATA_MOUSEWHEEL ) { imgui_ev.wheel.mouse_x /= scaling_factor; ... }
        }
        ImGui_ImplSDL3_ProcessEvent( &imgui_ev );
    }
}
```

完整链路：`SDL_PollEvent` → 坐标换算（窗口→buffer）→ `process_input`（窗口判定/NoMouse/÷缩放）→ `ImGui_ImplSDL3_ProcessEvent` → ImGui 输入队列 → 下一次 `NewFrame` 生效。

## 2. 捕获与分流（CDDA 的做法）

- 对外三个静态查询：`want_capture_mouse()` / `want_capture_keyboard()` / `want_text_input()`，即 `io.WantCaptureMouse` 等（`cata_imgui.cpp:712-725`），空 context 安全返回 false。
- **游戏侧按事件分流，不是"窗口一开就屏蔽"**（`sdltiles.cpp:6389-6467`）：

| 事件 | 条件 | 行为 |
| --- | --- | --- |
| `MOUSEMOTION` | 光标可见 | **光标显示逻辑无条件执行**；只有 `!want_capture_mouse()` 才产生 `MouseInput::Move` |
| `MOUSEBUTTONDOWN` | `!mouse.enabled` 或 `want_capture_mouse()` | break，不产生游戏输入 |
| `MOUSEBUTTONUP` | 同上 | break；否则按 L/R/X1/X2 产生对应释放事件 |
| `MOUSEWHEEL` | 同上 | break；否则 Up/Down |

- 要点：**鼠标移动（悬停高亮）与点击/滚轮（动作）区别处理**，移动只做"是否让游戏跟踪"、点击直接让给 ImGui；且光标可见性更新不受捕获判定影响。
- 键盘不做"吞事件"处理：CDDA 不拦 `KEYDOWN`，只在 Android 返回键分支用 `want_text_input()` + `clear_text_focus()` 协调焦点。
- `ImGuiConfigFlags_NoMouse` / `NoMouseCursorChange` 由玩家鼠标设置驱动（`sdltiles.cpp:384-410`）。

## 3. 坐标与缩放（两级换算）

CDDA 有**两个坐标系**：窗口像素 ↔ `display_buffer`（贴图，尺寸 = 窗口 / SCALING_FACTOR）。

1. **事件侧**：`convert_event_to_display_buffer_coords()`（`sdltiles.cpp:1307-1338`）对 motion/button/wheel 三类事件，用 `SDL_RenderCoordinatesFromWindow()` 把窗口坐标映射到 buffer 坐标（`sdltiles.cpp:1294-1304`，注释明确写着"use the SDL provided translation of scaling"）。
2. **渲染侧**：`new_frame()` 里显式设 `io.DisplaySize = display_buffer 尺寸`、`io.DisplayFramebufferScale = (1,1)`（`cata_imgui.cpp:596-609`），即 ImGui 的坐标系 == 绘制目标坐标系，不是窗口坐标系。
3. `process_input` 之后再 ÷ `scaling_factor`（UI 缩放与 SCALING_FACTOR 叠加时用）。

**CBN 现状的差异**：CBN 用 `SDL_SetRenderLogicalPresentation( renderer, W/sf, H/sf, STRETCH )`（`sdltiles.cpp:1418-1421`、`1618-1621`）让渲染器自己缩放，于是 ImGui 的**绘制**在逻辑坐标（W/sf），而**事件**仍是窗口像素、**从未换算**，`io.DisplaySize` 也从未设置（回落到后端取的 renderer 输出尺寸）。⇒ `SCALING_FACTOR ≥ 2` 时鼠标命中整体偏移 sf 倍；sf=1 时两者恰好重合，所以这个问题目前不暴露。

## 4. 与 CBN 当前代码逐项比较

| 维度 | CDDA master | CBN PR-1 现状 |
| --- | --- | --- |
| 事件入口 | `imclient->process_input()`，在 `CheckMessages` 顶部 | `cata_imgui::process_event(ev)`，在 `CheckMessages` 循环首行（`sdltiles.cpp:3228`） |
| 坐标换算 | 窗口→buffer + ÷scaling | **无**（直接用窗口像素） |
| `io.DisplaySize` | 显式设为 buffer 尺寸 | **未设置** |
| 帧驱动 | `ui_adaptor`/`window::draw()` 驱动 `new_frame/end_frame` | **由 `refresh_display()` 驱动**（主菜单实测 0.2 FPS） |
| 捕获查询 | 静态三函数，null-safe | 只有 `process_event` 内部用 `io.WantCaptureMouse/Keyboard` |
| 鼠标移动分流 | 光标逻辑无条件 + 捕获时才喂游戏 | 光标逻辑独立，但**无捕获判定**（一直喂游戏） |
| 鼠标按下 | `MOUSEBUTTONDOWN` 独立处理 | **未处理**（只有 `MOUSEBUTTON_UP`，`sdltiles.cpp:3418`） |
| 滚轮 | 捕获判定 + Up/Down | 有 Up/Down，**无捕获判定** |
| X1/X2 侧键 | 支持 | 不支持 |
| 鼠标开关 | `cata::options::mouse` + NoMouse 标志 | 无 |
| 窗口可见性 | `any_window_shown()` 才处理输入 | 只要 initialized 就处理 |
| 生命周期 | init/shutdown 成对 + 后端 active 闩锁 | `shutdown()` 存在但**无调用点** |

## 5. 对"F10 和鼠标没反应"的根因判断（基于上述对照）

1. **帧驱动缺陷（主因）**：CBN 的 ImGui 帧只在 `refresh_display()` 里推进，主菜单实测 **0.2 FPS ≈ 5 s/帧**。ImGui 的输入队列只在 `NewFrame` 时消费，且一次点击要求"先有一帧知道鼠标在此处悬停，再有帧看到按下"——5 s 一帧时单击几乎必然落空，表现为控件完全无反应。CDDA 由 UI 自己的 `new_frame/end_frame` 驱动，不受游戏重绘节奏影响，这是结构性差别。
2. **捕获判定滞后**：`WantCaptureMouse/Keyboard` 来自上一帧。CBN 用它决定是否吞事件，在 0.2 Hz 下判定严重滞后，吞与不吞都不可靠。
3. F10 的 `toggle()` 是立即执行的，但它只改 `g_open`；画面要等下一帧（≤5 s）才更新——用户按完看不到变化，自然判为"没反应"。
4. 事件通路本身是通的（`CheckMessages` 是唯一的 `SDL_PollEvent` 消费点，非 Android 分支下无第二个轮询点），所以**不是事件没送到**，而是"送到了但 UI 没有帧去消费/反馈"。

## 6. 分类结论

### 可以直接借鉴
1. `want_capture_mouse/keyboard/text_input` 三个静态查询函数（null-safe），供游戏侧按事件查询。
2. `process_input` 的三道闸：`any_window_shown()` → `NoMouse` 丢弃鼠标事件 → 拷贝事件后改坐标（不改游戏看到的那份）。
3. 事件分流粒度：鼠标移动只影响游戏悬停跟踪，鼠标点击/滚轮在捕获时直接让路；**光标可见性逻辑不受捕获影响**。
4. 每帧显式设 `io.DisplaySize` + `io.DisplayFramebufferScale=(1,1)`，让 ImGui 坐标系等于绘制目标坐标系。

### 需要适配 CBN
1. 坐标换算：CDDA 用 `SDL_RenderCoordinatesFromWindow` 做 window→buffer；CBN 走 `SDL_SetRenderLogicalPresentation`，应改为**同一来源的逆变换**——推荐照抄 CDDA 方案（保留窗口坐标系 + 显式换算 + 显式 DisplaySize），而不是继续依赖逻辑呈现自动缩放。
2. 事件类型：CDDA 用 `CATA_MOUSE*` 宏抽象，CBN 直接用 `SDL_EVENT_MOUSE_*`，需按 CBN 类型重写 switch。
3. `mouse.enabled` 选项：CDDA 有玩家级鼠标开关；CBN 无，可先用 `get_option("HIDE_CURSOR")` 或新增选项，暂不强求。
4. 帧驱动：把 `new_frame/end_frame` 从 `refresh_display()` 解耦，改成"ImGui 窗口打开时以固定频率（如 30–60 Hz）推进帧"，否则交互不可用。

### 不适合直接移植
1. `cataimgui::window` 类体系（`ui_adaptor`、filter box、`bounds` 缓存、font_reload、`abort_frame`）——依赖 CDDA 已重构的 UI 框架，CBN 无此基础。
2. `a11y`/`ImGuiConfigFlags_NoMouseCursorChange` 与本项目无关的光标策略。
3. Android 专属分支（`CATA_FINGER*`、快捷栏、安全区）整段跳过。
4. `cata_input_trail` / `cata_key_to_imgui` 键位映射表——CBN 的键盘路径不同，需按 CBN 的 `input_event` 体系另写。

### 当前已经实现（CBN）
1. `init/render/process_event/toggle/shutdown` 五件套 + `#if defined(TILES)` 隔离。
2. `imgui_impl_sdl3` + `imgui_impl_sdlrenderer3` 双后端，与游戏共用 renderer，画在 `RenderCopy` 之后、`RenderPresent` 之前。
3. F10 切换、`WantCaptureMouse/Keyboard` 的**雏形**判定、开窗/关窗时 `SDL_StartTextInput` 协调。
4. 字体图集在 backend init 之前构建（中文渲染已验证）。

### 当前缺失（CBN）
1. **帧驱动**：无独立帧循环，帧率绑死游戏重绘（主菜单 0.2 Hz）。
2. **坐标换算**：无 window→buffer 换算，无 `io.DisplaySize` 设置。
3. `MOUSEBUTTONDOWN` 处理；X1/X2 侧键。
4. 鼠标移动/滚轮的游戏侧捕获判定（现在 ImGui 打开时游戏仍会收到悬停与滚轮）。
5. `shutdown()` 调用点（PR-1 已知缺口）。
6. `any_window_shown()` 等价判断（现在 initialized 即处理事件）。

## 7. 建议的实施顺序（PR-1 收尾 / PR-2 前置）

1. 补 `shutdown()` 接入（3 行，退出清理前调用）。
2. 补 `io.DisplaySize` + `DisplayFramebufferScale`（对照 `cata_imgui.cpp:596-609`）。
3. 加 window→逻辑坐标换算（sf>1 才生效，sf=1 行为不变）。
4. 解耦帧驱动：窗口打开时按固定间隔调用 `refresh_display()` 或单独推进 ImGui 帧。
5. 鼠标分流改为 CDDA 式事件级判定（motion 只喂悬停、button/wheel 捕获时让路）。
6. 以上完成后，F10/鼠标/IME 三项验收才有意义，再谈 PR-2。

> 诚实标注：本报告的 CDDA 一侧结论全部来自 master 源码实际阅读；CBN 一侧结论来自本地源码与实测（0.2 FPS、截图、全量测试）。**"帧驱动导致点击落空"是基于 ImGui 输入语义 + 实测帧率的推断，尚未用插桩验证**——验证方法是给 `process_event` 与 `render` 加计数器/日志后重编。
