# 第1章 UGUI 整体架构

> 本章对应原书结构中的第1章（基础理论部分）。UGUI 是 Unity 官方推出的 UI 系统，全称 Unity UI，于 Unity 4.6 引入。这一章只做"热身"：用一页图让你知道 UGUI 整体长什么样、数据怎么流动、本书怎么读。具体机制（重建调度、RectTransform、Graphic、Canvas）从第 2 章起逐层展开。

---

## 1.1 为什么要先理解架构

在深入具体组件源码之前，先建立对 UGUI 整体架构的认知，有两个好处：

1. **遇到问题知道去哪找答案**：渲染问题是 Graphic 的职责还是 Canvas 的职责？
2. **避免"只见树木不见森林"**：不先建立架构认知，就会迷失在细节中。

本章只回答三个问题（每个问题都对应后面的专章）：

- UGUI 和 IMGUI 有什么本质区别？
- UGUI 的数据是怎么从 UI 描述变成屏幕像素的？
- UGUI 由哪些子系统组成，本书按什么顺序展开？

---

## 1.2 UGUI 是什么：五条设计取舍

UGUI 是一套 **保留模式（Retained Mode）** 的运行时 UI 系统：UI 结构以 GameObject + Component 的形式持久存在于场景中，而不是每帧用代码重绘。它的全部特性可以收敛成五条设计取舍，后面每一章都会反复用到：

1. **保留模式**：UI 结构持久存在，状态变化才更新，不做每帧重绘。
2. **组件化**：UI 元素是 GameObject 加 Component 的组合，可组合、可复用。
3. **延迟重建**：属性变化只打 Dirty 标记，统一在渲染前调度处理，把一帧内的多次修改合并成一次（第 4 章）。
4. **CPU 生成 Mesh**：UI 内容变化时在 CPU 侧重新生成顶点数据。这是 UGUI 灵活性的来源，也是主要性能成本——每帧改 Text 会持续触发重建（第 2、5、20 章）。
5. **自动合批**：系统按材质、纹理、Stencil 状态合并 DrawCall，合批条件很严格（第 9 章）。

这套取舍与第三方 UI 方案的关键差异：不少方案把网格生成放到 GPU 端，UGUI 相反，顶点始终由 CPU 生成。理解这一点，就能解释 UGUI 大多数性能问题为什么集中在 CPU 侧重建。

---

## 1.3 IMGUI vs UGUI：为什么要换

UGUI 之前，Unity 唯一的 UI 方案是 IMGUI（Immediate Mode GUI），即 `OnGUI()` 中的 `GUI.Button`、`GUI.Label` 等调用。它每帧执行绘制代码、UI 元素不保存状态，也没有 GameObject + Component 的复用结构。IMGUI 如今主要活跃在编辑器工具（Inspector、SceneView）中，运行时 UI 已被 UGUI 取代。

| 维度 | IMGUI | UGUI |
|------|-------|------|
| 渲染模式 | 即时（每帧重绘） | 保留（状态变化时更新） |
| 组织方式 | 代码控制 | GameObject + Component |
| 编辑方式 | 运行时代码 | 编辑器可视化 |
| 性能 | 每帧 CPU 密集 | Dirty 机制 + 延迟重建 |
| 适用场景 | 编辑器工具、调试面板 | 运行时 UI |

---

## 1.4 一页看懂数据流：从 UI 描述到屏幕像素

整个 UGUI 体系最重要的一句话：

> **所有 UI 最终都转换为 Mesh 数据提交给 GPU。**

在 UGUI 眼中，不存在"按钮"、"图片"、"文字"这些概念，只有顶点、三角形、UV 和颜色。一个 Image 在 GPU 眼里就是一个 Quad：

```
4 个顶点（四个角）→ 2 个三角形（6 个索引）→ 一个 Quad → 贴上纹理 → 一个 DrawCall
```

完整链路：

```
Image/Text 组件
  ↓ OnPopulateMesh(VertexHelper vh)（第 2/5 章）
Mesh（顶点 + 三角形 + UV + 颜色）
  ↓ canvasRenderer.SetMesh(mesh)（第 8 章）
CanvasRenderer（存储 Mesh）
  ↓ Canvas.BuildBatch()（引擎 native 方法，第 6 章）
合并 + 提交 DrawCall
  ↓
GPU：Vertex Shader → 光栅化 → Fragment Shader → FrameBuffer
```

这条链路**不是每帧都完整执行**：每帧固定执行的是输入轮询和渲染前的检查；事件分发、布局、重建、合批都是**条件触发**。以"UI 被输入修改后"的那一帧为例，完整时序是：

```
EventSystem.Update → 输入采集 → 命中检测 → 事件分发（第 12 章）
  → 回调修改 UI 属性 → 打 Dirty 标记，不立即重建（第 4 章）
  → Canvas 触发 willRenderCanvases
  → CanvasUpdateRegistry.PerformUpdate()
      ├── Layout Rebuild（第 11 章）
      ├── ClipperRegistry.Cull()（第 15 章）
      └── Graphic Rebuild → OnPopulateMesh → SetMesh（第 4/5 章）
  → Canvas.BuildBatch()（第 6/9 章）
  → GPU 像素输出
```

没有输入就不分发事件，没有 Dirty 就不重建，Canvas 数据没变就不重新合批——这就是保留模式省性能的方式。

---

## 1.5 三个逻辑层：组件 / 提交 / 执行

UGUI 的运行可以压成三个逻辑层（细节分别在第 4/5/6/8 章展开）：

| 层 | 代表组件 | 职责 |
|----|---------|------|
| 组件层 | Graphic 家族、Selectable、LayoutGroup | UI 结构定义、顶点数据生成、交互逻辑 |
| 提交层 | CanvasRenderer、Canvas、CanvasUpdateRegistry、LayoutRebuilder | 保存数据、调度重建、合批提交 DrawCall |
| 执行层 | Mesh / Material / Texture / Shader | GPU 侧执行 DrawCall，完成像素绘制 |

`Canvas.BuildBatch()` 是 CPU → GPU 的分界线：它之前是 C# 世界，之后是渲染管线世界。EventSystem 不在这条渲染链路上，它属于交互体系，按"输入采集 → 命中检测 → 事件分发"独立运转（第 12 章）。

---

## 1.6 系统组成与组件家族速览

UGUI 由三大体系加一组支撑系统组成：

| 体系 | 核心组件 | 负责什么 | 详见章节 |
|------|---------|---------|---------|
| 空间与布局 | RectTransform、LayoutGroup、LayoutRebuilder、ContentSizeFitter | 位置、尺寸、排列 | 第 3、11 章 |
| 渲染 | Graphic 家族、Canvas、CanvasRenderer | 顶点生成、合批、提交 | 第 2、5、6、8、9 章 |
| 交互 | EventSystem、InputModule、Raycaster、ExecuteEvents、Selectable | 输入采集、命中检测、事件分发 | 第 12、13 章 |
| 资源与图集 | Sprite、SpriteAtlas、动态图集 | 纹理组织与加载 | 第 14 章 |
| 裁剪与文本 | Mask、RectMask2D、Text/TMP、Shader | 裁剪、字形、着色 | 第 15、16、18 章 |
| 性能与工具 | Profiler、Frame Debugger、RenderDoc | 定位与优化 | 第 20、21、24 章 |

组件体系围绕几个抽象基类展开（完整继承树见第 5 章 5.2）：

```
UIBehaviour（所有 UI 组件的公共基类）
├── Graphic（抽象）→ MaskableGraphic → Image / RawImage / Text   ← 可视化
├── Selectable（抽象）→ Button / Toggle / Slider / Scrollbar / Dropdown / InputField  ← 交互
└── LayoutGroup（抽象）→ Horizontal / Vertical / GridLayoutGroup ← 布局
```

每个可见的 UI 元素都由三部分组成：RectTransform 决定位置尺寸（第 3 章），Graphic 子类生成顶点（第 5 章），CanvasRenderer 保存渲染数据（第 8 章）。可交互的元素再加一个 Selectable 子类（第 13 章）。

---

## 1.7 阅读路线：本书怎么组织

本书按"基础理论 → 渲染链路 → 性能与工具 → 工程实践"四部分、25 章组织。推荐两条路线：

- **新手路线（跟着链路走）**：第 2 章（从 RectTransform 定位 UI、再讲 UI 本质是 Mesh）→ 第 3 章（RectTransform 深入）→ 第 4 章（什么时候重建）→ 第 5 章（Graphic 怎么生成顶点）→ 第 6 章（Canvas 怎么合批提交）→ 第 8 章（CanvasRenderer）→ 第 9 章（DrawCall）。这一圈下来，UGUI 的渲染主线就通了。
- **专题路线（按需查）**：事件/交互看第 12/13 章，布局看第 11 章，文本看第 16 章，Mask 看第 15 章，性能诊断看第 20/21/24 章，自定义 Shader 看第 18/19 章。

章节地图、快速定位表和依赖图在 `README.md` 与 `agent.md` 中。

---

## 1.8 源码结构概览

UGUI 的 C# 源码以 Unity Package 形式发布。当前 main 分支的结构如下（旧版仓库如 2019.1 分支路径为 `UnityEngine.UI/UI/`，结构类似）：

```
com.unity.ugui/（GitHub: Unity-Technologies/uGUI, main 分支）
└── Runtime/UGUI/
    ├── UI/Core/                  ← 核心组件
    │   ├── Graphic.cs / MaskableGraphic.cs / Image.cs / RawImage.cs / Text.cs
    │   ├── Selectable.cs / Button.cs / Toggle.cs / Slider.cs / Scrollbar.cs / Dropdown.cs / InputField.cs
    │   ├── Mask.cs / RectMask2D.cs / StencilMaterial.cs / MaskUtilities.cs
    │   ├── CanvasUpdateRegistry.cs / GraphicRegistry.cs / GraphicRaycaster.cs
    │   ├── ScrollRect.cs / FontUpdateTracker.cs
    │   ├── Utility/              ← VertexHelper.cs（网格构建工具）
    │   ├── Layout/               ← 布局子系统
    │   │   ├── LayoutGroup.cs / LayoutRebuilder.cs / ContentSizeFitter.cs
    │   │   ├── HorizontalOrVerticalLayoutGroup.cs / GridLayoutGroup.cs
    │   │   └── CanvasScaler.cs
    │   ├── Culling/              ← 裁剪子系统（ClipperRegistry 等）
    │   └── VertexModifiers/      ← 顶点修改器（BaseMeshEffect / Shadow / Outline）
    └── EventSystem/              ← 事件系统，与 UI/Core 同级
        ├── EventSystem.cs / ExecuteEvents.cs / UIBehaviour.cs
        ├── InputModules/         ← BaseInputModule / StandaloneInputModule 等
        └── Raycasters/           ← BaseRaycaster / PhysicsRaycaster / Physics2DRaycaster
```

注意两点：

- `Canvas`、`CanvasRenderer`、`RectTransformUtility`、`UIVertex` 等属于 `UnityEngine.UIModule`，`RectTransform` 属于 `UnityEngine.CoreModule`——它们都是 Unity 引擎内置类型，不在 uGUI 仓库中。C# 侧只暴露部分 API，核心方法（如 `Canvas.BuildBatch`）标记 `[NativeMethod]`，由引擎 C++ 实现，看不到源码。
- `InputSystemUIInputModule`（新输入系统）不在 uGUI 仓库中，由 Input System 包提供。

---

## 1.9 本章总结

1. **UGUI 是保留模式的运行时 UI**：状态变化才更新，不做每帧重绘。
2. **五条设计取舍**：保留模式、组件化、延迟重建、CPU 生成 Mesh、自动合批。
3. **渲染本质**：所有 UI 最终都是 Mesh + Material + Texture，顶点由 CPU 生成。
4. **数据流**：UI 描述 → OnPopulateMesh 生成顶点 → SetMesh 存入 CanvasRenderer → Canvas.BuildBatch 合批提交 → GPU 像素输出；每一步条件触发。
5. **三个逻辑层**：组件层（生成数据）→ 提交层（调度 + 合批）→ 执行层（GPU）。
6. **阅读路线**：先"空间（RectTransform）→ 渲染（Mesh → 重建 → Graphic → Canvas）"走通主线，再按专题深入。

---

## 1.10 推荐的源码阅读路径

```
打开 Graphic.cs → 依次阅读：
  1. SetVerticesDirty()    ← 理解 Dirty 标记机制（第 4 章展开）
  2. SetMaterialDirty()    ← 同上
  3. Rebuild()             ← 理解重建流程的两个阶段
  4. OnPopulateMesh()      ← 虚方法，子类实现生成顶点（第 5 章展开）
  5. UpdateGeometry()      ← 理解 Mesh 如何提交给 CanvasRenderer
```

---

## 1.11 勘误汇总（对照 uGUI main）

| # | 严重程度 | 章节 | 原文声称 | 实际情况 |
|---|---------|------|---------|---------|
| 1 | 🟡 | 1.8 | `Canvas`、`CanvasRenderer`、`RectTransform` 均位于 `UnityEngine.CoreModule` | `Canvas`/`CanvasRenderer` 属 `UnityEngine.UIModule`，仅 `RectTransform` 在 `UnityEngine.CoreModule` |
| 2 | 🟡 | 1.8 | `VertexHelper.cs` 直接位于 `UI/Core/` | main 中位于 `UI/Core/Utility/` |
