 weasel 0.17.4 候选窗崩溃根因报告（Firefox 主线程 NULL 解引用）

日期：2026-10-17（v2，补充鼠标消息路径）
对象：rime/weasel 0.17.4（tag `0.17.4` = `9cc96e2`）安装版 `C:\Program Files\Rime\weasel-0.17.4\weasel.dll`
参照：GitHub issue #1857、6 份 Firefox minidump、官方 0.17.4 调试符号（PDB `80DF2EA8...`）
说明：**本报告所有行号均指 pristine 0.17.4**（补丁会使其下移）

---
# 开头先说：nightly（master）已经没有这个问题 —— 以及为什么

**升级到 nightly（`weasel-0.17.4.92.4aea197` = master 提交 `4aea197`）在源码层面已经走不到这条崩溃路径。** 这不是针对 #1857 的定点修复，而是 UI 重写（`dd3aea3 refactor(ui): reimprement ui by directcomposition`，3445 增 / 3611 删）顺手把它的三个必要条件都消掉了（逐点对齐见 §5）：

| 0.17.4 里让崩溃可能的东西 | nightly（master）现状 |
|---|---|
| `DoPaint()` 是全文件唯一没判 `m_layout` 的绘制入口 | `WeaselPanel.cpp:423-425` **函数开头**就有 `if (!m_pD2D \|\| !m_layout \|\| !m_pD2D->dc \|\| !m_pD2D->swapChain) return;`；同一条 `m_layout->GetAuxiliaryRect()`（L449）仍在，但在守卫之后 ✅ |
| 同一个 `WeaselPanel` 对象上会出现第二个窗口（第一个成孤儿，仍 thunk 回该对象） | `Create()`（L349-351）`if (m_hWnd) return !!m_hWnd;`，ATL thunk 换成自建 `WindowProc` + per-window `GWLP_USERDATA`（L365/L1171，`WM_NCDESTROY` 清零 L1192），`DestroyWindow()`（L161-172）会 `m_hWnd = nullptr` ✅ 架构上不再有"孤儿窗口仍派回对象" |
| 点选/划动直接解引用 `m_layout`；`RedrawWindow()` 直调 `DoPaint()` | 命中判定改走 `_GetInflatedCandRect()`，其 L604-608 自带判空；`RedrawWindow()` 换成 `InvalidateRect()`（`WeaselPanel.h:41`），直调路径消失 ✅ |

有一条值得单独记住：nightly 的 `Create()` **不建 layout**（没有 `WM_CREATE` 分支），layout 由随后的 `UI::Update()` → `Refresh()` 建，所以"窗口已存在、`m_layout == NULL`"在那边是**刚 Create 完那一瞬的正常状态** —— L424 那个守卫是**承重**的，不是装饰。（0.17.4 相反：`OnCreate` 里就 `Refresh()`。）

**但 nightly 也还缺三处**（本补丁有、master 没有，见 §5 表末行）：

- `_CreateLayout()` 没有兜底 `else`（未知 `layout_type` 仍会存 NULL，L266）；
- `_ResizeWindow()` 没有判空（L227-228）；
- TSF 侧 `_StartComposition()` 仍未检查 `RequestEditSession` 的返回值（`Composition.cpp:67` 的 `StartUI()` 仍在 `:70` 之前）—— 这是真正根治的那一层，**没有**放进本补丁（原因见 §6.3：改变 UI 元素建立时机，需真机回归）。

一句话：**升 nightly 能消掉本报告的崩溃与残留候选框，但那是"重写版自带守卫"，不是"修好了 0.17.4"**；0.17.4 该补还得补。

---

## 本仓库（fork）状态

- 这是 [`rime/weasel`](https://github.com/rime/weasel) 的 fork。**README 已由本报告取代**，上游原 README 保留为 [`README.upstream.md`](README.upstream.md)。
- 修复分支：[`fix/0.17.4-null_m_layout`](https://github.com/takomitkm/weasel/tree/fix/0.17.4-null_m_layout) —— 基于 tag `0.17.4`（`9cc96e2`）只加一个提交 [`e315d87`](https://github.com/takomitkm/weasel/commit/e315d87a5467a05f776af645ff6de2523c57b12a)：2 文件 **+40 −4**（`WeaselUI/WeaselPanel.cpp +26 −3`、`WeaselUI/WeaselUI.cpp +14 −1`）。
- 校验做到字节级：分支上 `WeaselUI/WeaselPanel.cpp` 的 blob `5713f68c54`、`WeaselUI/WeaselUI.cpp` 的 blob `40b9db12c4`，与本地逐字节一致；`git diff 0.17.4 e315d87` 与补丁文件 `weasel-0.17.4-null-layout-fix.patch`（4286 字节，sha256 `FF9183F1E1296797…`）完全相同；GitHub 自己算的 compare 也报 2 文件 +40 −4。
- 应用方式（补丁同时挂在 [Releases](https://github.com/takomitkm/weasel/releases) 附件里）：

  ```bash
  git -C <0.17.4 树> apply weasel-0.17.4-null-layout-fix.patch
  # 或直接切到上面那条分支
  ```

- **没编译、没真机验证**（开发机与校机都缺 MSVC / Windows SDK，见 §6.4）：补丁只做过文本级与 `git apply` 级校验。请把它当作"根因已由转储坐实、构建未验证"的修复。
- 之后怎么判断"修好了"：见 §7 的崩溃指纹与观察清单。

---

## 1. 结论

**崩溃点是 `WeaselPanel::DoPaint()` 里对 `m_layout` 的无保护解引用：**

| | |
|---|---|
| 文件 | `WeaselUI/WeaselPanel.cpp` |
| 0.17.4 行号 | **949**（`CRect auxrc = m_layout->GetAuxiliaryRect();`） |
| 机器码 | `weasel.dll+0x27147` = `mov rax, qword ptr [rcx]`，`rcx = [this+0x50] = m_layout = 0` |
| 异常 | `EXCEPTION_ACCESS_VIOLATION_READ`，`crash_address = 0x0`，主线程（6/6 份转储完全一致） |

**为什么会 NULL（两个缺陷叠加）：**

1. **`DoPaint()` 是全文件唯一一个没给 `m_layout` 加判定的绘制入口。** `m_layout` 在构造、`OnDestroy()`（L1108）、析构（L95）三处被置空；`MoveTo()`（L1121）作者已经写了 `if (!m_layout) return; // avoid handling nullptr in _RepositionWindow`，说明他们知道这个指针会为空——只是漏了 `DoPaint()`，而 `DoPaint()` 恰好是**系统消息触发的**那一个（`WM_PAINT`）。同类漏网还有三个鼠标处理器（见 §4.5）。
2. **panel 对象能收到属于"已经不该存在"的窗口的 `WM_PAINT`。** `WeaselPanel` 是 `UIImpl` 的成员对象（`WeaselUI.cpp:9`），生命周期远长于窗口；`UI::Create()` 复用同一个对象却**不检查窗口是否已存在**，0.17.4 走的又是 ATL 的 `CWindowImplBaseT::Create`（`ATLASSERT(m_hWnd == NULL)` 在 Release 里是空操作）。于是"第二次 Create"会把 `m_hWnd` 顶成新窗口，**旧窗口成为孤儿**：它还在屏幕上、它的 WndProc 还 thunk 到同一个 `WeaselPanel` 对象，但 `UIImpl` 的 `Show/Hide/Destroy` 全都只作用于 `m_hWnd`（新窗口），永远碰不到它。

**症状对应（与 #1857 和用户复现完全吻合）：**

- 「偶尔卡出一个候选框、无法消失」= **孤儿窗口**。它被画过候选词，之后没人能再 `Show/Hide` 它。
- 「切到微软英文输入法 → alt+tab → 必崩」= 切输入法触发 `WeaselTSF::Deactivate()`（`WeaselTSF.cpp:97`）→ `_cand->DestroyAll()`（L117）→ 销毁 `m_hWnd` 并把 `m_layout` 置空（`DestroyAll` 走 `Destroy(true)`，连整个 `UIImpl` 一起 `delete`）；随后 alt+tab 造成前台/合成变化，孤儿窗口被重绘 → 消息泵合成 `WM_PAINT` → `DoPaint()` → 解引用 NULL → Firefox 死。

---

## 2. 崩溃点证据

### 2.1 六份转储完全一致

`06-08 / 06-12 / 08-06 / 10-04 / 10-08 / 10-14` 的 dump 逐字段比对：

- `EXCEPTION_ACCESS_VIOLATION_READ`，`crash_address = 0x0`
- `crash_thread = 0`（浏览器主线程）
- weasel 偏移：`+0x27147 / +0x2A09A / +0x22CB3 / +0x2B6D2`（四帧固定）
- `code_id`、`debug_id = 80DF2EA812B94C3F9D5FCAAB14C242A81`（同一个二进制）

### 2.2 崩溃处指令序列（官方 PDB 符号化）

```
27136  cmp  byte ptr [rsi+0x30d], 0     ; hide_candidates
2713d  jne  ...                          ; if (!hide_candidates)
27143  mov  rcx, qword ptr [rsi+0x50]    ; m_layout
27147  mov  rax, qword ptr [rcx]         ; <- CRASH (rcx = 0)
2714e  call qword ptr [rax+0x18]         ; Layout::GetAuxiliaryRect()
27151  mov  rcx, qword ptr [rsi+0x50]
27155  mov  rax, qword ptr [rcx]
2715c  call qword ptr [rax+0x10]         ; Layout::GetPreeditRect()
```

`rsi` = `this`。寄存器实况（10-14 那份）：

```
Rsi(this) = 0x1514230a4e0     R13 = 0x1514230a7dc = this + 0x2FC   （= &rcw，CRect）
Rcx(m_layout) = 0             Rdx = 0
```

`this+0x30d` = `hide_candidates`（实测 0 = false），`this+0x50` = `m_layout`，`this+0x2fc` = `rcw` —— 与两条指令互相印证，成员偏移可靠。**6 份转储 `R13` 全部等于 `this+0x2FC`。**

### 2.3 调用栈（`.pdata` 函数边界 + 导出表定位，不依赖符号服务器）

```
firefox.exe!main                                    (+0x2351e)
 └─ … Firefox 事件循环 (xul.dll / mozglue.dll) …
    └─ user32!PeekMessageW                          user32+0x11948 / +0x119bf
       └─ user32+0x29bd8                            ← DispatchMessageW 区域
          └─ user32+0x152c6                         fn[0x14f90..0x15857) = CallWindowProcW 所在函数
             └─ atlthunk.dll+0x1028                 ← ATL 窗口过程蹦床
                └─ weasel+0x2b6d3                   fn[0x2b650..0x2b7c5) = WindowProc
                   └─ weasel+0x22cb4                fn[0x22930..0x22ce9) = ProcessWindowMessage (WeaselPanel.h:38)
                      └─ weasel+0x2a09b             fn[0x29e50..0x2a13e) = CDoubleBufferImpl<WeaselPanel>::OnPaint
                         └─ weasel+0x27147          DoPaint      ← 💥
```

要点：**这是 Firefox 自己的消息泵派发的正常 `WM_PAINT`**（`PeekMessage` → `DispatchMessage` → `CallWindowProc` → thunk → `WindowProc` → `OnPaint`），不是 `RedrawWindow()`、不是跨线程 `SendMessage`、不是 `PrintWindow`。

`WTL::CDoubleBufferImpl::OnPaint` 的符号行号是 `atlframe.h:3546`，对应 `else` 分支（`CPaintDC dc(pT->m_hWnd); ... pT->DoPaint(dc.m_hDC);`，即 `wParam == 0` 的真 `WM_PAINT`）；`WM_PRINTCLIENT` 走 3540 行那条。本仓库 `include/wtl/atlframe.h:3519-3522` 两条消息都映射到 `OnPaint`。

### 2.4 关键限制（诚实声明）

Firefox 写的是小型转储，只包含线程栈与少量内存。**`WeaselPanel` 对象本身（`this`）不在任何一份转储里**，所以"孤儿窗口"这一步无法在转储中直接观测，它是从源码推出的（见 §4）。转储能证明的是：`m_layout == NULL`、`hide_candidates == false`、窗口仍在正常收 `WM_PAINT`、以及派发路径。

---

## 3. 为什么 `m_layout` 会是 NULL

### 3.1 `m_layout` 的全部写入点（源码级穷举）

```
 53: : m_layout(NULL),                        // 构造
 94: delete m_layout;
 95: m_layout = NULL;                         // ~WeaselPanel()
108: if (m_layout != NULL) delete m_layout;    // _CreateLayout() 入口
128: m_layout = layout;                        // _CreateLayout() 出口
1107: delete m_layout;
1108: m_layout = NULL;                        // OnDestroy()
```

`_CreateLayout()` 的唯一调用者是 `Refresh()`（L159）。

### 3.2 `Refresh()` 的恒真式 —— 关键一步

```cpp
150:   bool inline_no_candidates =
151:       (m_style.inline_preedit && m_candidateCount == 0) && !show_tips;
152:   hide_candidates =
153:       inline_no_candidates || (margin_negative && !show_tips && !show_schema_menu);
157:   if (!hide_candidates || inline_no_candidates) {
158:     _InitFontRes();
159:     _CreateLayout();
```

`hide_candidates` 的定义就是 `inline_no_candidates || (...)`，所以条件 `!hide_candidates || inline_no_candidates` **恒为真**：

- `inline_no_candidates == true` → 第二项为真；
- `inline_no_candidates == false` → 第一项 `!hide_candidates == true`。

**结论：任何一次 `Refresh()` 都必然执行 `_CreateLayout()`；而 `hide_candidates` 也只在 `Refresh()` 里被写。**

于是：崩溃时 `hide_candidates == false`，说明最后一次 `Refresh()` 一定刚创建过 layout；而 `m_layout == NULL`，说明 **`m_layout` 是在那次 `Refresh()` 之后被清掉的** —— 只可能是 `OnDestroy()`（L1108）或 `~WeaselPanel()`（L95）。**窗口已经不在这个对象的名下了，可它还在画。**

这同时解释了"用户配置无关"：`margin_x/margin_y` 正负、`inline_preedit` 开关都不影响这个恒真式——与 #1857 里"两种设置都崩"一致。

### 3.3 一个次要的、与本次无关的空指针来源

`_CreateLayout()` 的 `if / else if` 链**没有兜底 `else`**（L112–127）；`layout_type` 是 IPC 反序列化字段，若对端是更新的构建（master 的枚举多了 `LAYOUT_VERTICAL_TEXT_FULLSCREEN`，值 5），本机会一条分支都匹配不上 → `m_layout = layout = NULL`。

**这不是本次崩溃的原因**：崩溃历史从 06-08 开始，而 nightly（含新枚举）是 10-12 才出现的，时间线对不上。但值得一并堵住（补丁已含）。

---

## 4. 为什么被销毁的窗口还能收到消息 —— 生命周期缺陷

### 4.1 对象与窗口的生命周期不同步

```
WeaselUI/WeaselUI.cpp (0.17.4)
  7: class weasel::UIImpl {
  9:   WeaselPanel panel;                       // ← 成员对象，不是指针
 11:   UIImpl(weasel::UI& ui) : panel(ui), shown(false) {}
 12:   ~UIImpl() {}                             // ← 空的：不销毁窗口

 85: bool UI::Create(HWND parent) {
 86:   if (pimpl_) {                            // ← 复用同一个 panel 对象
 87:     pimpl_->panel.Create(parent, 0, 0, WS_POPUP, ..., 0U, 0);   // ← 不检查 IsWindow()
 91:     return true;
 92:   }
 94:   pimpl_ = new UIImpl(*this);

105: void UI::Destroy(bool full) {
106:   if (pimpl_) {
108:     if (pimpl_->panel.IsWindow())
109:       pimpl_->panel.DestroyWindow();       // ← 只销毁 m_hWnd 记着的那个窗口
111:     if (full) { delete pimpl_; ... }        // ← full=false 时对象留着（m_layout 已为 NULL）
```

`CCandidateList` 持有 `std::unique_ptr<weasel::UI> _ui;`（`CandidateList.h:80`），即 `UI`/`UIImpl`/`WeaselPanel` 三者与输入会话同寿，而窗口只是其中一个字段。

### 4.2 第二次 `Create()` 会静默产生孤儿窗口

0.17.4 的 `WeaselPanel` 没有自定义 `Create`（`WeaselPanel.h` 只有消息映射，无 `Create` 声明），所以 `panel.Create(...)` 落在 ATL 的 `CWindowImplBaseT::Create` 上：

```
ATLASSERT(m_hWnd == NULL);        // Release 构建里是空操作
CreateWindowEx(...)
  └─ StartWindowProc(): pThis->m_hWnd = hWnd;            // ← 直接覆盖旧句柄
                        SetWindowLongPtr(hWnd, GWLP_WNDPROC, &m_thunk.cd);
```

同一个对象的两个窗口，`GWLP_WNDPROC` 是**同一个 thunk 地址**，两个窗口的消息都走进同一个 `WeaselPanel`。旧窗口从此：

- 没有任何字段指向它（`m_hWnd` 已被新窗口覆盖）；
- `UIImpl::Show/Hide/Refresh`、`UI::Destroy` 全都碰不到它；
- 它的 `WM_DESTROY` 仍会把 `m_hWnd`（现在是新窗口的！）置空、并把 `m_layout` 清成 NULL。

**这就是 #1857 里那个"卡住不消失的候选框"，也是崩溃的伏笔。**

### 4.3 孤儿窗口是怎么被造出来的（TSF 侧）

```
WeaselTSF/Composition.cpp
 73: void WeaselTSF::_StartComposition(com_ptr<ITfContext> pContext, BOOL fCUASWorkaroundEnabled) {
 76:   pStartCompositionEditSession.Attach(new CStartCompositionEditSession(...));
 78:   _cand->StartUI();                        // ← 先建窗口（StartUI → _MakeUIWindow → _ui->Create）
 81:   pContext->RequestEditSession(..., &hr);  // ← hr 被丢弃
     }

WeaselTSF/EditSession.cpp
 32:     if (_status.composing && !_IsComposing()) {
 33:       _StartComposition(...);              // ← 守卫是 _IsComposing()

WeaselTSF/Composition.cpp
419: BOOL WeaselTSF::_IsComposing() { return _pComposition != NULL; }
415: void WeaselTSF::_SetComposition(...)      { _pComposition = pComposition; }   // 由异步 session 调
411: void WeaselTSF::_FinalizeComposition()    { _pComposition = nullptr; }       // 由异步 session 调
```

窗口是**同步**建的，而决定"要不要建窗口"的 `_IsComposing()` 由**异步** edit session 里的 `_SetComposition/_FinalizeComposition` 驱动，且 `RequestEditSession` 的返回码被丢掉、`StartUI()`（`CandidateList.cpp:286-312`）里也没有"窗口已存在"的检查：

- edit session 没跑成（请求失败 / 上下文正在拆）→ 窗口已建、`_IsComposing()` 仍为 FALSE；
- 下一次 `_StartComposition` → 守卫放行 → `StartUI()` → `_ui->Create()` → **在活窗口上再 Create 一次** → 孤儿。

同一批常见路径都会清掉 `m_layout`：

```
WeaselTSF/CandidateList.cpp
229: void CCandidateList::Destroy()    { Show(FALSE); _DisposeUIWindow();    }   // full=false
235: void CCandidateList::DestroyAll() { Show(FALSE); _DisposeUIWindowAll(); }   // full=true → delete pimpl_
314: void CCandidateList::EndUI()      { ...; _DisposeUIWindow(); }              // 每次组合结束都走
WeaselTSF/Composition.cpp
402: void WeaselTSF::_AbortComposition(bool clear) { if (_IsComposing()) ...; _cand->Destroy(); }
WeaselTSF/WeaselTSF.cpp
 97: STDAPI WeaselTSF::Deactivate() { ...; _cand->DestroyAll(); }                // ← 切输入法走这里
```

### 4.4 症状与复现路径逐条对应

| 现象 | 机制 |
|---|---|
| 偶尔卡出一个候选框，不消失 | 孤儿窗口：被画过内容，且没有任何代码路径能再 `Show/Hide/Destroy` 它 |
| 切到微软英文输入法 | `ITfTextInputProcessor::Deactivate` → `WeaselTSF::Deactivate()` → `_cand->DestroyAll()` → `Destroy(true)`：销毁 `m_hWnd` 的窗口 + `delete pimpl_` → `~WeaselPanel` 里 `m_layout = NULL`（对象内存随后释放，但字段值仍在，`hide_candidates` 也仍是 false） |
| 再 alt+tab | 前台/合成变化让孤儿窗口产生无效区 → 消息泵合成 `WM_PAINT` → 派发到孤儿窗口的 WndProc（仍 thunk 到那个 panel 对象）→ `OnPaint` → `DoPaint` → `m_layout->GetAuxiliaryRect()` → NULL 解引用 → 浏览器进程结束 |
| 为什么 6 月到 10 月反复出现、且与 Firefox 版本无关 | 全部指向同一个二进制（同一 `debug_id`），是 weasel 侧缺陷，不是 Firefox 回归 |

### 4.5 第二条同类路径：鼠标消息（本次崩溃之外）

鼠标处理器的守卫只有 `hide_candidates`，而它**不会**在清 layout 时被复位（只有 `Refresh()` 写它），所以孤儿窗口收到鼠标消息时会直接踩 `m_layout`：

```
 257: OnLeftClickedUp    ── WM_LBUTTONUP
 261:   if (hide_candidates) { return 0; }        // ← 孤儿窗口上恒不成立
 274:   CRect rect = m_layout->GetCandidateRect((int)m_ctx.cinfo.highlighted);   // ← NULL
 285:         DestroyWindow();   // 若点到候选词：把 m_hWnd（可能是新窗口）也销毁，再清一次 m_layout
 288:       RedrawWindow();      // 没点到：RedrawWindow() 直接调 DoPaint()（不经消息泵）→ 同样崩
 295: OnLeftClickedDown  ── WM_LBUTTONDOWN
 299:   if (hide_candidates) { return 0; }
 309/343/356/369/383: m_layout->GetCandidateRect / offsetX / GetPrepageRect / GetNextpageRect / GetCandidateRect
 422: OnMouseMove        ── WM_MOUSEMOVE
 443:   CRect rect = m_layout->GetCandidateRect((int)i);                          // ← NULL
```

用户看到"卡住的候选框"时的第一反应就是**去点它或划过它**——那会走这条路径。本次 6 份转储的栈证明用户遇到的是 `WM_PAINT` 那条（栈上没有 `OnLeftClickedUp`），但这条必须一起修。

注：`RedrawWindow()`（L1086-1090）自己就是 `GetDC(); DoPaint(hdc); ReleaseDC();`，**所以给 `DoPaint()` 加守卫同时也封住了 `RedrawWindow()` 这条直调路径**。

---

## 5. 上游（master / nightly）现状 —— 源码级核对

nightly 资产 `weasel-0.17.4.92.4aea197-installer.exe` 的二进制里嵌着 `0.17.4.92.4aea197`；用 git 复现该串（`git rev-list 0.17.4..4aea197 --count` = **92**，`git rev-parse --short 4aea197` = **4aea197**）即可把该二进制绑定到提交 `4aea197`（= 当时 `origin/master` = tag `latest`，2026-10-12）。

**该版本对同一问题不是定点修复，而是整个 UI 重写（`dd3aea3` = `refactor(ui): reimprement ui by directcomposition`，3445 增 / 3611 删）时的"顺手带"：部分等价防住了，部分代码直接被换掉，还有两处漏着。** 逐点对齐本报告 §6.2 的补丁：

| 补丁点 | master `4aea197` 对应 | 判定 |
|---|---|---|
| #1 `DoPaint` 空判定 | `WeaselPanel.cpp:423-425` `DoPaint()` **开头** `if (!m_pD2D \|\| !m_layout \|\| !m_pD2D->dc \|\| !m_pD2D->swapChain) return;`；同一条 `m_layout->GetAuxiliaryRect()` 仍在（L449），但在守卫之后 | ✅ 等效（更强：函数级） |
| #7 不建第二个窗口 | `WeaselPanel.cpp:349-351` `BOOL WeaselPanel::Create(...) { if (m_hWnd) return !!m_hWnd;`；并把 ATL thunk 换成自建 `WindowProc` + per-window `GWLP_USERDATA`（L365/L1171，WM_NCDESTROY 时清零 L1192） | ✅ 等效（架构层面消掉"孤儿窗口仍派回对象"） |
| #4/#5 点选处理器 | 处理器被重写为 `OnLeftClickUp/OnLeftClickDown`，命中判定改用 `_GetInflatedCandRect()`，而它**自带**判空：`L604-608 if (!m_layout \|\| i < 0 \|\| i >= m_candidateCount \|\| i >= MAX_CANDIDATES_COUNT) { rc.SetRectEmpty(); return rc; }` | ✅ 等效（判空下沉到 helper） |
| #6 `OnMouseMove` | master 的 `OnMouseMove` 不再直接用 `m_layout`（走 `_GetInflatedCandRect`/`_CaptureRect`） | ✅ 等效 |
| `RedrawWindow()` 直调 `DoPaint` | master 的 `RedrawWindow()` 改成 `InvalidateRect(m_hWnd, nullptr, true)`（`WeaselPanel.h:41`）⇒ 直调路径消失 | ✅ 等效 |
| #2 `_CreateLayout` 兜底 | **没有**：`the<Layout> layout;` → `L266 m_layout = std::move(layout);`，匹配不上的 `layout_type` 仍然存 NULL（master 的 if/else 覆盖自身枚举全部 6 个值，故只有"未来新值"会中招） | ❌ master 没有 |
| #3 `_ResizeWindow` 守卫 | **没有**（`L227-228` 直接 `m_layout->GetContentSize()`；只从 `Refresh()` 里 `_CreateLayout()` 之后调用，实际安全） | ❌ master 没有 |
| #8 `~UIImpl` 销毁窗口 | master `~UIImpl() {}` **仍是空的**；但 `WeaselPanel::DestroyWindow()`（`L161-172`）会 `m_hWnd = nullptr`，`UI::Destroy(bool)`（`U:95-101`）在 `IsWindow()` 时先 `panel.DestroyWindow()` 再 `ReleaseAllResources()` | ⚠️ 机制不同，靠 Create 拒绝第二窗口闭环 |
| §6.3 TSF 侧根治 | **master 也没改**：`Composition.cpp:67 _cand->StartUI();` 仍在 `RequestEditSession`（`:70`）**之前**，`hr` 仍未被使用 | ❌ master 没有 |

两个值得记住的推论：

1. master 的 `Create()` **不建 layout**（没有 `OnCreate`/`WM_CREATE` 处理，`MsgHandler` 只在 `WM_PAINT/WM_DESTROY/WM_NCDESTROY/WM_MOUSEMOVE/...` 分支）；layout 由随后的 `UI::Update()` → `Refresh()` 建。也就是说**在 master 里"窗口已存在 + `m_layout == NULL`"是刚 Create 完那一瞬的正常状态** —— L424 那个守卫是**承重**的，不是装饰。这与 0.17.4 不同（0.17.4 在 `OnCreate` 里就 `Refresh()`，正常路径上 layout 与窗口同步建立）。
2. 因为 master 的枚举比 0.17.4 多一个值，而 `_CreateLayout()` 两版都没有兜底 else ⇒ **"nightly 服务端 + 0.17.4 TSF"这种混装会让 0.17.4 这边直接空指针崩溃**（§3.3）。补丁的兜底 `else` 正是防这个，而 nightly 自己防不了。

结论：**升级 nightly 能消掉本报告的崩溃与残留候选框（源码级）**，但那是"重写版自带守卫"，不是"修好了 0.17.4"；0.17.4 该补还得补，且补丁里有 nightly 也缺的两处加固。

---

## 6. 修复

### 6.1 交付物

```
补丁：S:\apps\llm\agentwork\_tmp\ffcrash\weasel-0.17.4-null-layout-fix.patch   (4286 字节, 2 文件, +40 -4)
工作树：S:\apps\llm\agentwork\weasel-0.17.4   （git worktree，detached at tag 0.17.4 = 9cc96e2，补丁已落上）
```

应用方式：

```bash
git -C <0.17.4 树> apply weasel-0.17.4-null-layout-fix.patch
```

已验证：`git apply --check --reverse` 在该工作树上通过（exit 0），即补丁与已改状态精确匹配，可干净套用到 pristine 0.17.4；两文件花括号平衡（`{`/`}` delta 均为 0）。

### 6.2 改动清单（6 处守卫 + 1 处兜底）

1. **`DoPaint()`（L948）—— 直接堵本次崩溃**：`if (!hide_candidates)` → `if (!hide_candidates && m_layout)`。
   守卫块跨 L948–L1061，`DoPaint()` 里**所有** `m_layout` 解引用都在块内（L949/950/955/959/971/989/999/1019/1031/1041/1044），块外的 `_LayerUpdate(rcw, memDC)`（L1062）不使用 layout。被守卫的调用链还包括 `_DrawCandidates` / `_DrawPreedit` / `_DrawPreeditBack` / `_HighlightText`（只从 `DoPaint` 调用）与 `RedrawWindow()`（直调 `DoPaint`）。语义等价于既有 `hide_candidates == true` 的跳过行为，不改变任何正常路径。
2. **`_CreateLayout()`（L128 前）—— 让 `m_layout` 永不为 NULL**：`if (layout == NULL) layout = new VerticalLayout(...);`（覆盖 §3.3 的未知枚举值）
3. **`_ResizeWindow()`（L99）**：`if (!m_layout) return;`（纵深防御；master 此处同样没有守卫）
4. **`OnLeftClickedUp()`（L261）**：`if (hide_candidates)` → `if (hide_candidates || !m_layout)`（§4.5）
5. **`OnLeftClickedDown()`（L299）**：同上
6. **`OnMouseMove()`（L426 前）**：函数入口加 `if (!m_layout) return 0;`
7. **`UI::Create()`（L86）—— 断掉孤儿窗口的产生**：
   ```cpp
   if (pimpl_) {
     if (pimpl_->panel.IsWindow()) {   // 对象只能管一个 HWND
       pimpl_->panel.Refresh();        // 复用活窗口（造型刚被 StartUI 更新过）
       return true;
     }
     pimpl_->panel.Create(...);
   ```
   与上游 master 的 `if (m_hWnd) return !!m_hWnd;` 同一意图，另外补了一次 `Refresh()`，因为 0.17.4 的 `UI::Update()` 在 `ctx/status` 未变时会提前返回，不刷新的话复用窗口可能留旧布局。
8. **`~UIImpl()`（L12）—— 兜底**：`if (panel.IsWindow()) panel.DestroyWindow();`
   保证"窗口绝不比对象活得久"，否则后续消息会被派发进已释放内存（`DestroyAll()` 的 `delete pimpl_` 就是这种情形）。

### 6.3 建议的、更根治的一处（**未包含**在补丁里）

`WeaselTSF/Composition.cpp:_StartComposition()`：窗口是在请求 edit session **之前**同步创建的，而 `RequestEditSession` 的 `HRESULT` 被丢弃。建议失败时回滚刚建的 UI：

```cpp
  _cand->StartUI();
  if (pStartCompositionEditSession != nullptr) {
    HRESULT hr;
    pContext->RequestEditSession(_tfClientId, pStartCompositionEditSession,
                                 TF_ES_ASYNCDONTCARE | TF_ES_READWRITE, &hr);
    if (FAILED(hr)) {
      // 该 edit session 不会再执行 → 组合不会真正开始，别把 UI 窗口留在那儿
      _EndUI();          // = _cand->EndUI()：EndUIElement(uiid) + 销毁窗口
    }
  }
```

或更彻底：把 `StartUI()` 移到 `CStartCompositionEditSession::DoEditSession()` 里（组合真正开始时才建 UI）。没有放进补丁，因为它改变 UI 元素建立时机，需要真机回归（尤其要确认 `ITfUIElementMgr::BeginUIElement/EndUIElement` 的配对与 `uiid` 时序）。

### 6.4 未验证项（明确标注）

- **没有编译**：本机与校机目前都没有 MSVC / Windows SDK（校机账号 `YiZhidi` 无管理员权限；开发机 `Visual Studio\2022` 与 Windows Kits 是空壳）。补丁只做了文本级与 `git apply` 级校验。
- **没有复现**：孤儿窗口出现的确切时机（哪一次 `RequestEditSession` 失败）属于源码推断，需要在真机打点（`UI::Create()` 复用分支、`_StartComposition()` 的 `hr`、`OnCreate/OnDestroy`）才能坐实。
- 崩溃本身的四个事实（崩溃地址 `+0x27147`、`m_layout == NULL`、`hide_candidates == false`、正常 `WM_PAINT` 派发路径）是转储实测，不是推断。

---

## 7. 之后怎么判断"修好了"

崩溃指纹（便于以后自动筛 dump）：`weasel.dll` 的 `EXCEPTION_ACCESS_VIOLATION_READ @ 0x0`，RIP = 该 DLL 基址 `+0x27147`，`Rcx == 0`，主线程，栈上能见到 `atlthunk.dll+0x1028`。换版本后偏移会变，用"`weasel` 模块 + 访问地址 0 + DoPaint 帧"判定更稳。

观察清单：

1. 是否还出现"卡住不消失的候选框"（孤儿窗口的肉眼特征，比崩溃更早出现）；
2. `C:\Users\takomi\AppData\Roaming\Mozilla\Firefox\Crash Reports\pending\` 是否不再新增同指纹 dump；
3. 对照实验：只装 nightly（含两处守卫）观察一周，与带本补丁的 0.17.4 对比；注意 nightly 是 UI 重写，不能用它的表现单独判定本补丁的效果。

---

## 附：补丁全文

完整补丁见 `weasel-0.17.4-null-layout-fix.patch`（同目录）。改动分布：

```
WeaselUI/WeaselPanel.cpp  @@ -97,6 +97,8    _ResizeWindow        空守卫
                          @@ -125,6 +127,15 _CreateLayout        兜底 else
                          @@ -258,7 +269,10 OnLeftClickedUp      || !m_layout
                          @@ -296,7 +310,8  OnLeftClickedDown    || !m_layout
                          @@ -423,6 +438,10 OnMouseMove          入口守卫
                          @@ -945,7 +964,11 DoPaint              && m_layout
WeaselUI/WeaselUI.cpp     @@ -9,7 +9,12     ~UIImpl              销毁窗口
                          @@ -84,6 +89,14    UI::Create           复用活窗口
```

---

## 附：本报告用到的证据文件

| 路径 | 说明 |
|---|---|
| `C:\Users\takomi\AppData\Roaming\Mozilla\Firefox\Crash Reports\pending\*.dmp/.extra` | 6 份崩溃转储（同指纹） |
| `S:\apps\llm\agentwork\_tmp\ffcrash\store\weaselx64.pdb\80df2ea812b94c3f9d5fcaab14c242a81\` | 0.17.4 官方调试符号（`debug_id` 与转储一致） |
| `S:\apps\llm\agentwork\_tmp\ffcrash\stackwalk.py` | 裸解析 minidump 栈 + 模块归属 |
| `S:\apps\llm\agentwork\_tmp\ffcrash\symlocal.py` | 用 `.pdata` + 导出表做本地符号化（不依赖符号服务器） |
| `S:\apps\llm\agentwork\_tmp\ffcrash\panelstate.py` | 从转储读 `WeaselPanel` 字段（结论：对象内存不包含在转储中） |
| `S:\apps\llm\agentwork\weasel` | rime/weasel clone（`0.17.4` 与 `4aea197` 两个版本都在此对比） |
| `S:\apps\llm\agentwork\weasel-0.17.4` | 0.17.4 worktree（补丁已落上） |
