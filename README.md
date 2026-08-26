# cosmic-term
COSMIC terminal emulator, built using [alacritty\_terminal](https://docs.rs/alacritty_terminal) that is provided by the [alacritty](https://github.com/alacritty/alacritty) project. `cosmic-term` provides bidirectional rendering and ligatures with a custom renderer based on [cosmic-text](https://github.com/pop-os/cosmic-text).

The `wgpu` feature, enabled by default, supports GPU rendering using `glyphon`
and `wgpu`. If `wgpu` is not enabled or fails to initialize, then rendering falls
back to using `softbuffer` and `tiny-skia`.

## Color Schemes

Custom color schemes can be imported from the `View -> Color schemes...` menu item.
You can find templates for color schemes in the [color-schemes](color-schemes) folder.

## HACK变更说明

以下是本仓库在上游基础上做出的改动：

### 分割布局改为「每个标签页独立」（架构重构）

- 原实现中，水平/垂直分割作用于**窗口级**的全局窗格网格，所有标签页共享同一分割布局，新建标签页也会继承分割状态。
- 现已反转为常见终端模拟器的行为：**每个标签页拥有自己独立的窗格网格**（`terminal.rs` 中的 `Tabs` / `TerminalTab` / `PaneContent`）。
  - 分割（Ctrl+Alt+D / Ctrl+Alt+R）只影响当前标签页；
  - 新建标签页（Ctrl+Shift+T）获得全新的单窗格布局，不受其他标签页影响；
  - 切换回某个标签页时，恢复该标签页上次聚焦的窗格；
  - 关闭窗格 → 关闭最后一个窗格时关闭标签页 → 关闭最后一个标签页时关闭窗口，链路与上游一致；
  - 终端事件路由由 `(pane, entity)` 改为 `(标签页实体, 窗格)` 二元组，后台标签页的事件仍能正确定位。

### 窗口级标签栏

- 标签栏从「每个窗格内部」移至**菜单栏下方**，整窗唯一，单标签页时也显示；
- 样式沿用原窗格内标签栏（主题 base 底色），支持点击切换、X 关闭与**拖拽重排**；
- 拖拽重排依赖 fork 版 libcosmic（`hack` 分支）新增的拖拽图标功能（见 libcosmic 仓库的 HACK变更说明）。

### 其他行为改进

- 分割线颜色：分割缝隙使用当前终端主题的背景色填充，与终端融为一体（COSMIC 内置主题故意不设终端背景色，此时回退到 `WINDOW_BG_COLOR`，即应用主题底色）。
- 工作目录继承：开启「新标签和窗口使用当前目录」设置后，**分割新建的终端**同样继承当前目录（此前仅新建标签页/窗口继承）。
- 标签页创建时带 `.closable()`，修复标签栏上没有 X 关闭按钮的问题。

### 构建配置

- `libcosmic` 依赖指向 [jmjoy/libcosmic](https://github.com/jmjoy/libcosmic.git) 的 `hack` 分支（含上述拖拽图标等改动），而非上游 pop-os 仓库。
- 同时保留 `[patch.'https://github.com/pop-os/libcosmic']` 段，将 pop-os 来源的 `libcosmic` / `cosmic-config` / `cosmic-theme` 一并重定向到该 fork 的 `hack` 分支——因为 `cosmic-files` 等 git 依赖会在内部引用 pop-os 的 libcosmic，必须统一到同一来源，否则会出现两份不同版本的 libcosmic 导致类型冲突。
