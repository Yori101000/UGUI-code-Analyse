# UGUI 源码级原理分析

> 本书是 Unity UGUI 的源码级原理分析：从组件源码到渲染管线，再到工程实践，共四部分 25 章。行为守则见 `AGENTS.md`，导航与已知问题见 `agent.md`。

## 读者定位

本书面向**有 UGUI 使用经验、想弄清底层机制的中级 Unity 开发者**，不是零基础入门读物。

**适合你，如果：**

- 已经用 UGUI 搭过界面，知道 Canvas / Image / Text / LayoutGroup 大致各自做什么
- 遇到过 DrawCall 偏高、UI 卡顿、Mask 断批、文本性能这类问题，想知道根因而不只是"怎么绕过去"
- 想在动手改代码之前，先理解 UGUI 为什么这样设计

**不适合你，如果：**

- 第一次接触 UGUI —— 建议先照官方手册搭两三个界面，有了直观印象再回来
- 想要 API 速查 —— Unity 官方 Scripting API 更合适
- 想要一份"照着抄就能用"的组件库 —— 本书的代码以讲清机制为目的（见下节）

**前置知识：** C# 基础、Unity 编辑器基本操作，以及顶点 / 三角形 / 纹理 / DrawCall 这些图形学概念的基本印象。**不要求会写 Shader**——第 18 章从 `UI/Default` 逐行讲起。

## 关于本书的代码

- 正文中的 C# 片段多数标注「简化示意」：**删去了编辑器分支、空值校验、属性包装等噪音**，只保留讲解所需的主干。真实实现请对照 uGUI 仓库 main 分支，两者在结构上一致、在细节上不同。
- Shader 示例以讲清原理为主，直接复制进工程可能需要补 `#include` 等样板代码。
- 所有源码事实以 [uGUI main 分支](https://github.com/Unity-Technologies/uGUI/tree/main) 为准。`Canvas`、`CanvasRenderer`、`RectTransform`、`UIVertex` 等属引擎内置类型，**不在该仓库中**，以 Unity 官方文档为准；凡依赖行为反推的结论，正文会标注「行为推断」，引用时不要升级为"源码证实"。

## 目录结构

```
UGUI-code-Analyse/
├── README.md                  ← 本文件：总目录 + 章节依赖图
├── AGENTS.md                  ← Agent 行为守则
├── agent.md                   ← Agent 导航手册（章节地图、已知问题）
├── 第一部分_基础理论/          ← 第 1~9 章：UGUI 是什么、怎么运转
├── 第二部分_渲染链路/          ← 第 10~19 章：从 Mesh 到像素
├── 第三部分_性能与工具/        ← 第 20~21 章：性能模型与调试工具
├── 第四部分_工程实践/          ← 第 22~25 章：架构、动画、案例与选型
└── _阅读指南/                 ← 源码映射导读、勘误记录
```

## 章节目录

### 第一部分：基础理论（第 1~9 章）

| 章 | 主题 | 一句话 |
|---|------|--------|
| 01 | UGUI 整体架构 | 纯引入：IMGUI vs UGUI、五条设计取舍、数据流总览、系统组成、阅读路线 |
| 02 | UI 的本质：从图形到网格 | 从 RectTransform 入手，所有 UI 都是 Mesh：UIVertex / VertexHelper |
| 03 | RectTransform 核心机制 | 锚点、轴心、矩形计算与坐标系 |
| 04 | UI 更新与重建系统 | Dirty 标记、CanvasUpdateRegistry、重建调度 |
| 05 | Graphic 系统 | 顶点生成、Rebuild 流程、OnPopulateMesh |
| 06 | Canvas 系统 | 渲染模式、排序、BuildBatch、多 Canvas |
| 07 | CanvasScaler 分辨率适配 | 三种缩放模式与 scaleFactor |
| 08 | CanvasRenderer 机制 | 数据容器、SetMesh/SetMaterial、Native 桥梁 |
| 09 | UI 批处理与 DrawCall | 合批条件、断批原因、Frame Debugger 诊断 |

### 第二部分：渲染链路（第 10~19 章）

| 章 | 主题 | 一句话 |
|---|------|--------|
| 10 | UI 与渲染管线 | Built-in/URP/HDRP 下的 UI 渲染、Camera Stack、RenderQueue |
| 11 | Layout 布局系统 | LayoutGroup、ContentSizeFitter、ScrollRect 深度解析 |
| 12 | EventSystem 事件系统 | 输入采集、Raycaster、ExecuteEvents 分发 |
| 13 | Selectable 与交互组件体系 | 状态机、过渡方式、导航、Button/Toggle/Slider |
| 14 | UI 资源与图集系统 | Sprite/SpriteAtlas、动态图集、资源生命周期 |
| 15 | Mask 与裁剪机制 | Stencil Mask、RectMask2D、断批与选型 |
| 16 | 文本渲染系统 | Legacy Text 链路、TMP 的 SDF 与缓存架构 |
| 17 | UI Mesh 扩展机制 | IMeshModifier、Shadow/Outline 顶点复制 |
| 18 | UI Shader 机制 | UI-Default 解析、Blend、Stencil、自定义 Shader |
| 19 | UI 特效实现 | 渐变/流光/灰化/描边/发光/溶解/毛玻璃 |

### 第三部分：性能与工具（第 20~21 章）

| 章 | 主题 | 一句话 |
|---|------|--------|
| 20 | UI 性能分析 | 五个性能维度：Batch、材质、Mask、顶点、Overdraw |
| 21 | 调试与分析工具 | Frame Debugger、UI Profiler、RenderDoc、自定义工具 |

### 第四部分：工程实践（第 22~25 章）

| 章 | 主题 | 一句话 |
|---|------|--------|
| 22 | UI 架构设计 | 分层模式、UIManager、UI 栈、生命周期 |
| 23 | DOTween 原理与 UGUI 集成 | 对象池、委托驱动、与 Rebuild 的交互 |
| 24 | 综合案例与实战 | 背包、虚拟列表、弹窗、Profiler 实战三案例 |
| 25 | UI Toolkit 对比分析 | UGUI vs UI Toolkit 的架构与选型 |

## 章节依赖图

```
第一部分：基础理论
  01 整体架构 ← 全书入口
  ├─ 02 UI本质（从 RectTransform 入手 → Mesh）─→ 05 Graphic ─→ 08 CanvasRenderer ─→ 09 批处理
  ├─ 03 RectTransform ─→ 04 更新与重建 ─→ 09
  ├─ 06 Canvas（含 07 CanvasScaler）

第二部分：渲染链路
  10 渲染管线 ← 依赖 09
  11 Layout     ← 依赖 03、04
  12 EventSystem（输入，独立）
  13 Selectable  ← 依赖 12
  14 图集        ← 依赖 09
  15 Mask        ← 依赖 09、18
  16 文本/TMP    ← 依赖 05、18
  17 Mesh 扩展   ← 依赖 05
  18 Shader      ← 依赖 09、15
  19 特效        ← 依赖 17、18

第三部分：性能与工具
  20 性能分析 ← 依赖 09~19 的代价模型
  21 调试工具

第四部分：工程实践
  22 架构设计 ← 依赖 01
  23 DOTween   ← 依赖 04、05
  24 综合案例  ← 依赖 11、12、20
  25 UI Toolkit 对比
```

## 阅读路径

- **系统阅读**：01 → 02~09 → 10~19 → 20~21 → 22~25，章节顺序即链路顺序
- **性能专题**：01、04、09、15、18、20、21、24
- **交互专题**：01、12、13，列表滚动再看 11（ScrollRect）
- **源码深挖**：配合 uGUI 仓库 https://github.com/Unity-Technologies/uGUI/tree/main 逐章对照

### 章节依赖不是线性的

编号按主题组织，但实际依赖是网状的（见上方依赖图）。有两处编号与依赖倒挂，阅读时留意：

- **第 15 章 Mask 用到 Stencil，而 Stencil 在 shader 层面的完整解释在第 18 章。** 15 章自带了够用的 Stencil 讲解（15.1.1~15.1.4），先读哪一章都可以。
- **第 9 章合批的三个关键变量分散在别处**：材质从哪来见第 5 章 `materialForRendering`，Stencil 状态见第 15 章，Shader Keyword 见第 18 章。第 9 章给的是结论清单，细节需要回查这三章。

正文中大量「详见第 N 章」是交叉参考，**首轮阅读不必跟着跳转**——按编号顺序读完，再回头补需要的部分。

## 维护约定

- 所有源码事实以 uGUI 官方仓库 main 分支为准，不确定时先核实再写（`AGENTS.md` 强制）。
- 章节顺序以文件名编号为准（四部分内 01~25）；任何内容修改后必须同步 `agent.md`。
- 全库 UTF-8 无 BOM；PowerShell 读取加 `-Encoding UTF8`。
