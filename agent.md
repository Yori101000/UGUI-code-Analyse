# agent.md：项目导航手册（Agent Readme）

> 本文档面向后续阅读 / 修改本仓库的 AI Agent，也适合人类协作者快速上手。
> 核心目标：让任何 Agent 用最短时间理解**本仓库是什么、怎么组织、有哪些坑、怎么动手改**。
> 行为守则见 `AGENTS.md`：其中要求任何内容修改后都必须重新审视并同步本文件，保证其时效性。

---

## 0. 一句话总结

这是一个纯 Markdown 的中文知识库，主题是 **Unity UGUI 源码级原理分析**：四部分 25 章 + 阅读指南，共 34 个 `.md` 文档（约 585KB）。
它不是代码项目：没有工程代码、没有 LICENSE。仓库入口是 `README.md`（读者定位 + 总目录 + 章节依赖图 + 阅读路径）和本文件（导航手册）；Claude Code 另有 `CLAUDE.md` 作为上手说明，内容是 `AGENTS.md` + 本文件的约束摘要，不构成独立规则来源。

**读者定位已确定为「有 UGUI 使用经验的中级开发者」**（见 `README.md` 开头）：不做初学者化改造——不新增「怎么读源码」导读章、不加动手实验环节、不做「简化代码 vs main 真实代码」并排展示。此结论已评估并否决，后续不必重复提议。

---

## 1. 仓库构成

```
UGUI-code-Analyse/
├── README.md                   ← 读者定位 + 总目录 + 章节依赖图 + 阅读路径
├── AGENTS.md                   ← Agent 行为守则（源码参考规则、维护义务）
├── agent.md                    ← 本文件：Agent 入口导航
├── CLAUDE.md                   ← Claude Code 专用上手说明（约束摘要 + 模板/勘误/编号约定）
├── 第一部分_基础理论/          ← 第 01~09 章
├── 第二部分_渲染链路/          ← 第 10~19 章
├── 第三部分_性能与工具/        ← 第 20~21 章
├── 第四部分_工程实践/          ← 第 22~25 章
└── _阅读指南/                  ← 源码映射导读、审查/审计报告、勘误记录、历史存档
    ├── UGUI源码导读.md            ← 按 01~25 编号的源码映射
    ├── 全库审查报告（2026-08-14）.md ← 最新：47 项问题台账（P0 未修）
    ├── 源码一致性审计报告.md       ← 2026-08 首次审计（实际覆盖 9 章）
    └── 第1-8章源码勘误.md          ← ⚠ 已过期，勿引用（见下方第 4 节）
```

- **章节顺序的唯一可信来源是文件名编号**（四部分内 `第NN章`，NN = 01~25）。
- 全部文件为 UTF-8 无 BOM。在 PowerShell 中读取必须加 `-Encoding UTF8`，否则中文显示为乱码。

---

## 2. 主题地图（四部分 25 章）

| 章 | 主题 | 关键内容 |
|---|------|---------|
| 01 | UGUI 整体架构 | 纯引入：IMGUI vs UGUI、五条设计取舍、数据流总览、三层职责、系统组成、阅读路线、源码结构 |
| 02 | UI 的本质：从图形到网格 | 从 RectTransform 入手、UIVertex、VertexHelper、Image 网格模式、内存与 GC 代价 |
| 03 | RectTransform 核心机制 | 锚点/轴心、offsetMin/Max、sizeDelta、anchoredPosition、RectTransformUtility |
| 04 | UI 更新与重建系统 | Dirty Flag、CanvasUpdateRegistry、PerformUpdate 阶段、完整重建链路 |
| 05 | Graphic 系统 | 继承层级、三种 Dirty、Rebuild 流程、OnPopulateMesh、VertexHelper 使用要点、自定义 Graphic 路径 |
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

1. **章节模板**（2026-08-14 起全库统一，新增/修改章节必须遵守）：

   ```
   # 第N章 主题
   > 引言 blockquote          ← 直接讲本章主题，不要写"本章对应原书第 X 章"
   ## 概述
   ## N.1 / N.2 ...           ← 编号小节
   ---
   ## 本章总结                 ← 标题固定这五个字，不加编号
   ---
   ## 源码阅读路径             ← 标题固定这六个字，不加编号、不加"推荐的"
   ---
   ## 勘误汇总                 ← 可带括号补充，如「勘误汇总（对照 uGUI main）」
   ```

   三个收尾段一律 `##` 级、顺序固定为 **本章总结 → 源码阅读路径 → 勘误汇总**，段间用 `---` 分隔。章内小节自己的"源码阅读路径"（如 11.7.11）不受此约束，保持 `###` 级留在原处。
2. **引言写法**：部分章节开头为「本章对应原书结构中的第 N 章…」，这里的"原书"指重构前的旧章节体系，不能据此判断当前章节号。
3. **呈现形式**：大量使用表格、ASCII 框图、简化后的 C# 源码片段、性能数字对比。
4. **勘误汇总表**：列格式为 `# | 严重程度 | 章节 | 原文声称 | 实际情况`，严重程度用 🔴 严重 / 🟡 中等 / 🟢 轻微。修正内容时保留"原文 vs 实际"对照，不静默改写。
5. **历史说明**：正文保留的"原文第 N 章 / 原文 N.M 节"表述指重构前编号，属纠错记录，不是当前章节号。

### 源码引用基线（重要，避免"权威性错误"）

- 开源部分：GitHub 仓库 `Unity-Technologies/uGUI`，**以 main 分支为准**。当前 main 为 `com.unity.ugui` 包结构（`Runtime/UGUI/UI/Core/`、`Runtime/UGUI/EventSystem/`）；旧分支（如 2019.1）为 `UnityEngine.UI/UI/`。
- **存疑仲裁规则（AGENTS.md 强制）**：对任何 UGUI 机制、类、方法、路径或行为不确定对错时，先到 https://github.com/Unity-Technologies/uGUI/tree/main 核实再作答或修改，不得凭记忆或二手资料下结论。
- **引擎内置、不在 uGUI 仓库中的组件**：`Canvas`、`CanvasRenderer`、`RectTransformUtility`、`UIVertex`、`TextGenerator`、`RenderMode` 等属 `UnityEngine.UIModule`（`RectTransform` 属 `UnityEngine.CoreModule`），核心方法标记 `[NativeMethod]`（如 `Canvas.BuildBatch`），C# 侧看不到实现。`UIVertex` 在 Unity 6 起含 `prevPosition` 字段，uGUI main 经 TEXCOORD4 通道读写。
- **TextMeshPro 不在 uGUI 仓库中**：TMP 是独立包 `com.unity.textmeshpro`（源码仓库 Unity-Technologies/TextMeshPro），源码路径为 `Scripts/Runtime/...`。
- `EventSystem` 在 uGUI 仓库中与 `UI/Core` **同级**（`EventSystem/` 目录），不在 Core 内部。
- `UI-Default.shader` 等 UI Shader 是引擎内置资源，不在 C# 仓库中。
- 凡依赖"反推 / 官方文档 / Frame Debugger 观察"的结论，正文通常会注明验证方式，引用时不要升级为"源码证实"。

---

## 4. 当前已知问题（2026-08 重构后）

> ✅ **全库审查报告（2026-08-14）：47 项中已修复 44 项**，清单与逐条落点见 `_阅读指南/全库审查报告（2026-08-14）.md`。P0（跨章矛盾 12 项 + 编造 API 5 项 + 元文档失真 5 项）与 P1（代码缺陷 9 项 + 性能数字 5 项）已全部处理完毕。
>
> **本轮统一的口径**（后续改动不要再改回去）：
>
> - **RectMask2D**：`SetClipRect()` → `canvasRenderer.EnableRectClipping()` → Shader 的 `UNITY_UI_CLIP_RECT` + `_ClipRect`。**不改顶点、不建材质实例，但会断批**（同一裁剪矩形内可合批，跨 RectMask2D 断批）；**不减少 Overdraw**（裁剪写在片元着色器内部）。权威章节：第 15 章。
> - **`Graphic.material`**：getter 无副作用、不实例化；触发 `SetMaterialDirty()` 的是 setter。**`Graphic` 没有 `sharedMaterial`**。权威章节：第 20 章 20.2。
> - **Mask 合批**：`StencilMaterial.Add()` 带缓存，同 Stencil 参数共享实例，**同深度子元素之间可以合批**；断批来自不同深度与 Hierarchy 中的状态交替。权威章节：第 15 章 15.1.7。
> - **Stencil 与片元着色器时序**：不能一概说"Stencil 在片元着色器之后"，Early Stencil 取决于 GPU 与配置。权威章节：第 18 章 18.4.3。
> - **Dirty 触发**：uGUI 中没有 `SetColor()` / `SetSprite()` / `SetText()` 这类方法，全是属性 setter。权威章节：第 4 章 4.6。
>
> **剩余 3 项未做**（属全库性重排或新增内容，需独立提交）：F5 收尾体例不统一、F6 章节颗粒度失衡、G 内容增补。

1. **颗粒度不均**：第 16、19、20 章在 30KB 量级，第 07、21 章等偏小；合并产生的章节内部结构仍在演进。
2. **勘误表已改为如实说明**：原先 07/13/20/21/22/23/24/25 章的占位行「本章无源码级勘误记录」会被误读为"已核查"，现已改为列出「已核查项 / 未核查项」，或明确写"本章尚未做逐条源码核查"。新增勘误请继续沿用「原文声称 / 实际情况」对照格式。
3. **历史 / 过期文档，勿当现状引用**：
   - `_阅读指南/整体结构分析（历史存档）.md` —— 早期重构规划，已标注存档。
   - `_阅读指南/第1-8章源码勘误.md` —— ⚠ **已过期且部分结论与现状相反**，文首已加 🔴 警示（列出 5 条失效原因）。仅作历史追溯，**不得作为验证依据**。
   - `_阅读指南/源码一致性审计报告.md` —— 结论仍有效，覆盖面声明已更正为实际的 01/03/04/06/08/09/16/17/19 九章；未覆盖部分由 `全库审查报告（2026-08-14）.md` 补充。
4. ~~重构未提交~~ **已提交**：四部分目录、合并与编号统一的重构已作为独立提交落库（`4dd6929`），工作区干净；后续内容勘误（如 02/05 章 VertexHelper 按 main 修正）单独提交。
5. **源码一致性审计（2026-08）**：已对照 uGUI main 与官方文档修正第 01/03/04/06/08/09/16/17/19 章的同类问题（引擎归属、编造 API、UIVertex 大小、TMP 归属、TextGenerator 方法名等），完整清单见 `_阅读指南/源码一致性审计报告.md`。**注意其覆盖面窄于声明**（见上方第 3 条）。
6. **新增 `CLAUDE.md`（2026-08）**：Claude Code 上手说明，仅汇总既有约束（源码基线、agent.md 同步义务、UTF-8 无 BOM、章节模板、勘误优先、编号陷阱）。若日后调整规则，须与 `AGENTS.md` / 本文件同步修改，避免三处口径不一致。
7. **第 1/2/5 章结构重排（2026-08）**：第 1 章改为纯引入；第 2 章从 RectTransform 入手再讲 Mesh；第 5 章 5.7 与第 2 章去重（内部结构指向 2.4~2.7）。`_阅读指南/UGUI源码导读.md` 已按新编号重写。

---

## 5. 给 Agent 的操作指引

### 阅读 / 问答类任务

- 用户说"第 N 章"时确认是否指文件名编号；回复中统一使用文件名编号（01~25）并给出文件链接。
- 引用源码事实时区分三种来源：uGUI 仓库 C# 源码 / 引擎内置不可见源码 / 文档或行为反推。
- 回答性能类问题时，优先查 04（重建）、09（批处理）、15（Mask）、18（Shader）、20（性能）、21（工具）、24（案例）。
- 以下话题全书已统一口径（2026-08-14 修复），**权威章节如下，其余章节只做交叉引用，不要在别处另立说法**：
  - **RectMask2D 的机制与断批** → 第 15 章（08/09/14/20/24 章均已对齐）
  - **`Graphic.material` 语义、材质实例从哪来** → 第 20 章 20.2
  - **Mask 内子元素能否合批** → 第 15 章 15.1.7
  - **Stencil 与片元着色器时序** → 第 18 章 18.4.3
  - **哪些操作触发 Dirty** → 第 4 章 4.6（全是属性 setter，没有 `SetColor()` 之类的方法）
  - **`UI/Default` 的完整结构** → 第 18 章 18.1

### 修改类任务

- 保持 **UTF-8 无 BOM** 编码与现有文件名格式（`第NN章_主题.md`），不移动文件所属部分目录除非任务需要。
- 遵循章节模板与写作风格（见第 3 节），不要引入新的结构约定。
- 修正源码级错误时：优先在对应「勘误汇总」表中追加行（原文声称 / 实际情况），而不是静默改写正文。
- 改动前用 `git status` 确认工作区状态；本仓库历史提交均为中文单条描述。
- **修改完成后必须回到本文件同步章节地图、快速定位表与已知问题清单**（`AGENTS.md` 中的强制维护义务），并在最终回复中说明同步结果。

### 新增 / 重构类任务

- 新章节按四部分内的序号取号，需同步 `README.md` 目录表与依赖图。
- 全库性改动（编号、目录、合并）应作为独立提交，与内容修改分离，便于 review 和回滚。
- **拆分/合并章节文件、调整章节编号属于高影响改动，动手前先征求用户意见**，不要自行决定。说明时给出：为什么章内重排不够、拆分方案、以及需要连带修改的范围（全库「详见第 N 章」交叉引用、`README.md` 目录表与依赖图、`_阅读指南/UGUI源码导读.md` 的章节条目、各章勘误表中的章号）。用户确认后再执行，并作为独立提交。

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

- ~~F5 收尾体例统一~~ **已完成（2026-08-14）**：全库 25 章的收尾段已统一为 `## 本章总结` → `## 源码阅读路径` → `## 勘误汇总`，模板见第 3 节。
- **F6 颗粒度失衡（进行中）**：已完成第一批章内重排——19 章抽取 Shader 公共骨架（19.0）消除 6 处重复、11.8 压缩为 Layout 视角并指向 20/24 章、10 章合并 Overdraw 四节（原 10.6.3~10.6.6 → 新 10.6.3）与 URP 重复段（原 10.7.1/10.7.2/10.7.4 并入 10.7 引子）；25 章从 6.4KB 补到 12.5KB。当前跨度 32.5KB（10 章）~ 6.8KB（07 章）。
  **注意：去重不等于变小** —— 10/11 章删掉重复后体积反而微增，因为换成了更密的独有内容。继续做时以"消除重复、补齐缺口"为目标，不要为压体积删有效内容。
  剩余可做：05 章 5.3 渲染链路与第 2/8 章重叠；16 章 Font.textureRebuilt 链路仍有重复（该章勘误 #3 自述在六个小节重复）。
  **若判断必须拆成新文件、需要重新编号，先向用户说明理由并征求意见**（代价见第 5 节）。
- 内容增补方向（报告 G 节）：~~25 章过薄~~ 已补（渲染路径独立性、自定义渲染能力差距、排序/输入边界、本章总结）；仍缺 `CanvasGroup` 专门讲解、`Image` 的 Sliced/Tiled/Filled 与 Simple 详略不对称、`Dropdown`/`InputField` 仅在 13 章列名。
- **不做的方向**：初学者化改造（新增「怎么读源码」导读章、动手实验、简化代码与 main 真实代码并排展示）已评估并否决，定位保持中级开发者，见第 0 节。
