# agent.md：项目导航手册（Agent Readme）

> 本文档面向后续阅读 / 修改本仓库的 AI Agent，也适合人类协作者快速上手。
> 核心目标：让任何 Agent 用最短时间理解**本仓库是什么、怎么组织、有哪些坑、怎么动手改**。
> 行为守则见 `AGENTS.md`：其中要求任何内容修改后都必须重新审视并同步本文件，保证其时效性。

---

## 0. 一句话总结

这是一个纯 Markdown 的中文知识库，主题是 **Unity UGUI 源码级原理分析**：四部分 25 章 + 阅读指南，共 31 个 `.md` 文档（约 490KB）。
它不是代码项目：没有工程代码、没有 LICENSE。仓库入口是 `README.md`（总目录 + 章节依赖图）和本文件（导航手册）。

---

## 1. 仓库构成

```
UGUI-code-Analyse/
├── README.md                   ← 总目录 + 章节依赖图 + 阅读路径
├── AGENTS.md                   ← Agent 行为守则（源码参考规则、维护义务）
├── agent.md                    ← 本文件：Agent 入口导航
├── 第一部分_基础理论/          ← 第 01~09 章
├── 第二部分_渲染链路/          ← 第 10~19 章
├── 第三部分_性能与工具/        ← 第 20~21 章
├── 第四部分_工程实践/          ← 第 22~25 章
└── _阅读指南/                  ← 源码映射导读、勘误记录、历史存档
```

- **章节顺序的唯一可信来源是文件名编号**（四部分内 `第NN章`，NN = 01~25）。
- 全部文件为 UTF-8 无 BOM。在 PowerShell 中读取必须加 `-Encoding UTF8`，否则中文显示为乱码。

---

## 2. 主题地图（四部分 25 章）

| 章 | 主题 | 关键内容 |
|---|------|---------|
| 01 | UGUI 整体架构 | IMGUI vs UGUI、设计特点、职责分层（组件/提交/执行）、渲染本质、系统组成全景、继承总览、一帧流转、源码结构 |
| 02 | UI 的本质：从图形到网格 | UIVertex、VertexHelper、Image 四种网格模式、内存与 GC 代价 |
| 03 | RectTransform 核心机制 | 锚点/轴心、offsetMin/Max、sizeDelta、anchoredPosition、RectTransformUtility |
| 04 | UI 更新与重建系统 | Dirty Flag、CanvasUpdateRegistry、PerformUpdate 阶段、完整重建链路 |
| 05 | Graphic 系统 | 继承层级、三种 Dirty、Rebuild 流程、OnPopulateMesh、自定义 Graphic 路径 |
| 06 | Canvas 系统 | 三大职责、三种 RenderMode、BuildBatch、排序规则、AdditionalShaderChannels |
| 07 | CanvasScaler 分辨率适配 | ConstantPixelSize / ScaleWithScreenSize / ConstantPhysicalSize、scaleFactor |
| 08 | CanvasRenderer 机制 | "提交者而非生成者"、SetMesh/SetMaterial、[NativeMethod]、裁剪支持 |
| 09 | UI 批处理与 DrawCall | 合批精确条件、Batch 断裂原因、Frame Debugger 诊断、常见误区 |
| 10 | UI 与渲染管线 | Built-in/URP/HDRP 行为、Camera Stack、RenderQueue、SRP Batcher、Overdraw（含原 URP/HDRP 章） |
| 11 | Layout 布局系统 | ILayoutElement/Controller、LayoutGroup、ContentSizeFitter、循环依赖、ScrollRect 源码解析（含原 ScrollRect 章） |
| 12 | EventSystem 事件系统 | InputModule、GraphicRaycaster、ExecuteEvents 委托表、点击生命周期、新输入系统 |
| 13 | Selectable 与交互组件体系 | SelectionState 状态机、三种过渡、导航系统、Button/Toggle/Slider |
| 14 | UI 资源与图集系统 | Sprite vs Texture、SpriteAtlas、动态图集、资源加载与释放 |
| 15 | Mask 与裁剪机制 | Stencil Mask、RectMask2D、断批原理、实战选型决策树 |
| 16 | 文本渲染系统 | 字符到像素链路、Font.textureRebuilt、Legacy Text 流程、TMP 的 SDF/缓存/材质（含原 TMP 深析章） |
| 17 | UI Mesh 扩展机制 | IMeshModifier、BaseMeshEffect、Shadow/Outline 顶点复制、多特效叠加 |
| 18 | UI Shader 机制 | UI-Default.shader 逐行解析、Blend/Overdraw、Stencil 块、自定义模板 |
| 19 | UI 特效实现 | 渐变/流光/灰化/描边/发光/溶解/毛玻璃：原理 + Shader + 性能 |
| 20 | UI 性能分析 | Batch 中断、材质实例化、Mask 代价、顶点膨胀、Overdraw、工程级优化 |
| 21 | 调试与分析工具 | Frame Debugger、UI Profiler、RenderDoc、4 个自定义调试工具 |
| 22 | UI 架构设计 | UI 分层模式、UIManager、UI 栈、UIBase 生命周期标准化 |
| 23 | DOTween 原理与 UGUI 集成 | 对象池、委托驱动、Sequence、对 Rebuild/性能的影响、Kill 时机 |
| 24 | 综合案例与实战 | 背包系统、虚拟列表、弹窗系统、Profiler 实战三案例（含原 Profiler 章） |
| 25 | UI Toolkit 对比分析 | 架构/布局/事件/性能对比、选型建议、共存策略 |

### 快速定位（"想查什么 → 看哪章"）

| 问题 | 章节 |
|---|---|
| UI 怎么变成 Mesh / 顶点数据 | 02、05、17 |
| 为什么 UI 修改会触发重建、怎么避免 | 04、20 |
| 分辨率适配 / CanvasScaler | 07 |
| 合批条件 / 断批原因 | 09、15、18 |
| Mask / RectMask2D 怎么选 | 15 |
| 事件怎么分发 / 自定义事件 | 12 |
| 布局 / 虚拟列表 / 对象池 | 11、24 |
| 文本性能 / TMP | 16 |
| 自定义 UI Shader / 特效 | 18、19 |
| 性能诊断工具与实战 | 21、24 |
| UI 架构 / UIManager / 弹窗 | 22、24 |
| 渲染管线（URP/HDRP）/ UI Toolkit | 10、25 |

---

## 3. 内容约定与写作风格

1. **章节模板**：引言 blockquote → 概述 → 编号小节（`N.M`）→ 源码阅读路径 → 本章总结 →（修正过的章节）勘误汇总。
2. **引言写法**：部分章节开头为「本章对应原书结构中的第 N 章…」，这里的"原书"指重构前的旧章节体系，不能据此判断当前章节号。
3. **呈现形式**：大量使用表格、ASCII 框图、简化后的 C# 源码片段、性能数字对比。
4. **勘误汇总表**：列格式为 `# | 严重程度 | 章节 | 原文声称 | 实际情况`，严重程度用 🔴 严重 / 🟡 中等 / 🟢 轻微。修正内容时保留"原文 vs 实际"对照，不静默改写。
5. **历史说明**：正文保留的"原文第 N 章 / 原文 N.M 节"表述指重构前编号，属纠错记录，不是当前章节号。

### 源码引用基线（重要，避免"权威性错误"）

- 开源部分：GitHub 仓库 `Unity-Technologies/uGUI`，**以 main 分支为准**。当前 main 为 `com.unity.ugui` 包结构（`Runtime/UGUI/UI/Core/`、`Runtime/UGUI/EventSystem/`）；旧分支（如 2019.1）为 `UnityEngine.UI/UI/`。
- **存疑仲裁规则（AGENTS.md 强制）**：对任何 UGUI 机制、类、方法、路径或行为不确定对错时，先到 https://github.com/Unity-Technologies/uGUI/tree/main 核实再作答或修改，不得凭记忆或二手资料下结论。
- **引擎内置、不在 uGUI 仓库中的组件**：`Canvas`、`CanvasRenderer`、`RectTransform`、`RectTransformUtility`、`TextGenerator` 等，位于 `UnityEngine.CoreModule`，核心方法标记 `[NativeMethod]`（如 `Canvas.BuildBatch`），C# 侧看不到实现。`UIVertex` 同样不在仓库内，属 `UnityEngine.UIModule` 引擎类型（Unity 6 起含 `prevPosition` 字段，uGUI main 经 TEXCOORD4 通道读写）。
- `EventSystem` 在 uGUI 仓库中与 `UI/Core` **同级**（`EventSystem/` 目录），不在 Core 内部。
- `UI-Default.shader` 等 UI Shader 是引擎内置资源，不在 C# 仓库中。
- 凡依赖"反推 / 官方文档 / Frame Debugger 观察"的结论，正文通常会注明验证方式，引用时不要升级为"源码证实"。

---

## 4. 当前已知问题（2026-08 重构后）

1. **颗粒度不均**：第 16、19、20 章在 30KB 量级，第 07、21 章等偏小；合并产生的章节内部结构仍在演进。
2. **《`UGUI源码导读.md`》仍按旧编号编写**：已加"结构更新"提示（含新旧对应与 TMP/URP/ScrollRect/Profiler 并入说明），待按新结构重写。
3. **模板不完整**：部分章节（尤其合并/新增章节）缺「勘误汇总」或「源码阅读路径」段。
4. **历史文档**：`_阅读指南/整体结构分析（历史存档）.md` 已过时，仅作追溯，勿当现状。
5. ~~重构未提交~~ **已提交**：四部分目录、合并与编号统一的重构已作为独立提交落库（`4dd6929`），工作区干净；后续内容勘误（如 02/05 章 VertexHelper 按 main 修正）单独提交。

---

## 5. 给 Agent 的操作指引

### 阅读 / 问答类任务

- 用户说"第 N 章"时确认是否指文件名编号；回复中统一使用文件名编号（01~25）并给出文件链接。
- 引用源码事实时区分三种来源：uGUI 仓库 C# 源码 / 引擎内置不可见源码 / 文档或行为反推。
- 回答性能类问题时，优先查 04（重建）、09（批处理）、15（Mask）、18（Shader）、20（性能）、21（工具）、24（案例）。

### 修改类任务

- 保持 **UTF-8 无 BOM** 编码与现有文件名格式（`第NN章_主题.md`），不移动文件所属部分目录除非任务需要。
- 遵循章节模板与写作风格（见第 3 节），不要引入新的结构约定。
- 修正源码级错误时：优先在对应「勘误汇总」表中追加行（原文声称 / 实际情况），而不是静默改写正文。
- 改动前用 `git status` 确认工作区状态；本仓库历史提交均为中文单条描述。
- **修改完成后必须回到本文件同步章节地图、快速定位表与已知问题清单**（`AGENTS.md` 中的强制维护义务），并在最终回复中说明同步结果。

### 新增 / 重构类任务

- 新章节按四部分内的序号取号，需同步 `README.md` 目录表与依赖图。
- 全库性改动（编号、目录、合并）应作为独立提交，与内容修改分离，便于 review 和回滚。

---

## 6. Git 背景

- 分支 `master`；2026-08 重构（四部分目录化、合并章节、统一编号 01~25）已提交（`4dd6929`），当前工作区干净。
- 历史要点：早期 25 章结构 → 完整重编（基础理论 + 渲染链路 + 工程实践）→ 新增 26~29 章 → **2026-08 重构**：四部分目录化、合并 4 对章节（10+27、11+16、17+26、25+29）、统一全库编号 01~25、新增 README、归档过时规划文档。

---

## 7. 外部参考资源

| 资源 | 用途 |
|---|---|
| https://github.com/Unity-Technologies/uGUI | UGUI C# 源码（以 main 分支为准） |
| UnityBuiltinShaders / 安装目录 `Data/Resources/BuiltinShaders/UI-Default.shader` | 默认 UI Shader 源码 |
| Frame Debugger（Window → Analysis） | 逐 DrawCall 看断批原因 |
| Profiler UI Module / Timeline | 定位 Rebuild、Layout、Batch 瓶颈 |
| RenderDoc | GPU 侧抓帧分析 |
| TextMeshPro、URP/HDRP、UI Toolkit、DOTween 官方文档 | 第 16/10/25/23 章涉及 |

---

## 8. 建议的后续改进（尚未实施）

- 按新结构重写 `_阅读指南/UGUI源码导读.md`（补齐 02、07、13、16、20、21、24、25 等章节映射）。
- 为合并/新增章节补全「勘误汇总 / 源码阅读路径」模板段。
- 评估拆分偏大章节（19 特效、20 性能）或继续精简重复内容。
