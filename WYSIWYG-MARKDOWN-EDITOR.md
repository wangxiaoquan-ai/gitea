# Gitea 仓库 Markdown 文件 WYSIWYG 编辑改造方案（已确认版）

> 基线：release/v28（稳定维护线，tag v28.1.0 所在分支）

## 1. 背景与目标
- Gitea 无官方插件系统，仓库文件在线编辑当前使用 CodeMirror 6（纯源码模式）。
- 目标：**仅针对仓库内 Markdown 文件（.md / .markdown）的在线编辑**，替换为 Typora 式所见即所得（WYSIWYG）编辑器，可读可写，保存后提交到仓库。
- 不支持 GFM 高级方言（mermaid、@提及、任务列表等），仅基础语法：标题、加粗/斜体、列表、引用、代码块、链接、图片。
- 已确认决策：内部 fork 自建维护 ｜ 选型 Vditor ｜ 仅仓库文件编辑场景 ｜ 单一所见即所得模式，不做源码切换兜底。

## 1.1 前车之鉴（已核实，非从零摸索）
- **GitLab Content Editor**：Tiptap 2.0 + ProseMirror 三层架构（工具栏 UI / 状态管理 / Markdown 双向序列化器），2021 年 6 月随 GitLab 14.0 首发，早期仅支持基础 Markdown（与本方案范围重合），至今持续维护（2024 年仍在升级 tiptap 依赖），GitLab 17.7 增加默认编辑器偏好设置。代码位于 CE 仓库 `app/assets/javascripts/content_editor/`。
- **GitLab Static Site Editor**：现成 WYSIWYG 库（ToastUI）替换 textarea，范围同样限定基础语法、不做图片上传视觉化，与本方案取舍同构——印证"基础语法 + 现成库替换 = 低风险组件替换"。
- 本方案等价于该路线，将库替换为 Vditor（中文输入法更稳）。

## 2. 技术路线
- 数据层、后端 API、Git 提交链路：**零改动**。Vditor 编辑后产出纯 Markdown 文本，写回原表单字段，沿用现有提交流程（编辑页 → 提交说明 → 创建提交）。
- 服务端渲染与 XSS 消毒：不变；编辑态显示与服务端渲染样式对齐（见第 5 节）。
- 改造点（文件编辑链路，非评论框链路）：
  - `web_src/js/features/codeeditor.js` —— 文件编辑器初始化逻辑（当前 CodeMirror 6）
  - `templates/repo/editor/edit.tmpl` —— 编辑页模板（含隐藏的 textarea 表单字段）
- 切换策略：按文件名扩展名判断——`.md/.markdown` 加载 Vditor（mode: wysiwyg），其余文件类型维持 CodeMirror 6 不变。

## 3. Vditor 集成要点
- [ ] 编辑页按扩展名分流初始化 Vditor
- [ ] 提交时将 Vditor `getValue()` 回写至原 textarea，保证表单契约不变
- [ ] 页面离开/内容变更提示（beforeunload）接回
- [ ] Vditor 依赖以 npm 包引入（`vditor`），经现有 Webpack 构建链打包
- [ ] 懒加载：仅编辑 .md 文件时动态 import，避免拖慢其他页面

## 4. 功能接回清单
- [ ] 图片上传/粘贴插入（复用仓库附件上传接口，或退化为让用户填图片 URL）
- [ ] 快捷键冲突检查（与 Gitea 页面全局快捷键）
- [ ] 编辑器设置（如 Gitea 原有的缩进、换行符选项）：.md 场景下由 Vditor 自身行为接管，需在文档中向用户说明差异

## 5. 个性化样式
- Vditor 编辑区域样式与 Gitea 渲染态 `.markup` 排版对齐（标题间距、代码块、引用块样式），保证"所见 = 提交后所得"。
- 读 Gitea CSS 变量（`--color-*`）覆盖 Vditor 主题变量，暗色主题（gitea-dark / arc-green）自动适配。
- 自定义样式优先走零侵入路径（`custom/templates/` + `[ui]` 注入 CSS），减少升级 rebase 冲突面。

## 6. 风险控制
- 改动隔离：仅影响"仓库文件编辑页"，不触碰评论、Wiki、Issue/PR 等其他编辑器（它们仍走原 ComboMarkdownEditor 链路）。
- 兜底手段：fork 分支维护 + 版本 tag，出问题回退构建产物即可，数据无损（提交物始终是纯 Markdown）。
- 重点实测项：中文输入法（IME 组合输入稳定性）、大图/长文档性能、暗色主题样式。
- 升级维护：改动集中在 `codeeditor.js` 与编辑页模板，跟随上游 release 分支 rebase 时冲突面小。

## 7. 里程碑建议
1. M1：fork 基线 + Vditor 原型（仅 .md 编辑页，能编辑能保存）
2. M2：图片插入、快捷键、beforeunload 接回 + 样式对齐（含暗色）
3. M3：IME 专项测试 + 长文档性能测试 + 内部试用验收
