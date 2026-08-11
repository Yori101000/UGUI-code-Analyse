# 第1章 UGUI 整体架构

> 本章对应原书结构中的第1章（基础理论部分）。UGUI 是 Unity 官方推出的 UI 系统，全称 Unity UI，于 Unity 4.6 引入。理解 UGUI 的整体架构，是后续章节深入分析各个子系统的前提。

---

## 1.1 为什么要先理解架构

在深入具体组件源码之前，先建立对 UGUI 整体架构的认知，有两个好处：

1. **遇到问题知道去哪找答案**：渲染问题是 Graphic 的职责还是 Canvas 的职责？
2. **避免"只见树木不见森林"**：不先建立架构认知，就会迷失在细节中。

本章回答几个问题：
- UGUI 和 IMGUI 有什么本质区别？
- UGUI 由哪些子系统组成，它们怎么协作？
- UGUI 的三层架构怎么运转？
- UGUI 的渲染本质是什么？
- 一个 UI 元素一帧内经历了什么？

---

## 1.2 IMGUI vs UGUI

Unity 在 UGUI 之前，唯一的 UI 方案是 IMGUI（Immediate Mode GUI），即 `OnGUI()` 函数中的 `GUI.Button`、`GUI.Label` 等调用。

### 1.2.1 IMGUI：即时模式

```csharp
// IMGUI：每帧执行绘制代码
void OnGUI() {
    if (GUI.Button(new Rect(10, 10, 100, 50), "Click Me")) {
        Debug.Log("Button clicked");
    }
}
```

IMGUI 的特征：
- **每帧重绘**：`OnGUI()` 每帧被调用，所有 UI 元素每帧重新绘制
- **无状态**：UI 元素不保存状态，每帧从头绘制
- **缺乏组件化**：没有 GameObject + Component 的概念，UI 元素不可复用
- **定位为编辑器工具**：IMGUI 在 Unity 编辑器中大量使用（Inspector、SceneView 等），运行时 UI 中已基本被 UGUI 取代

### 1.2.2 UGUI：保留模式（Retained Mode）

```csharp
// UGUI：声明结构，系统维护
// 将 Button 拖到场景中 → 系统只在状态变化时重建
```

UGUI 的特征：
- **保留模式**：UI 结构在场景中持久存在，不是每帧重绘
- **GameObject + Component**：每个 UI 元素是一个 GameObject，上面挂载多个 Component
- **可视化编辑**：通过 SceneView 和 Inspector 拖拽编辑
- **自动批处理**：系统自动合并 DrawCall

### 1.2.3 核心差异

| 维度 | IMGUI | UGUI |
|------|-------|------|
| 渲染模式 | 即时（每帧重绘） | 保留（状态变化时更新） |
| 组织方式 | 代码控制 | GameObject + Component |
| 编辑方式 | 运行时代码 | 编辑器可视化 |
| 性能 | 每帧 CPU 密集 | Dirty 机制 + 延迟重建 |
| 适用场景 | 编辑器工具、调试面板 | 运行时 UI |

---

## 1.3 UGUI 的设计特点

上一节的对比可以收敛成 UGUI 自己的几条设计取舍，后面每一章都会反复用到：

**保留模式**，UI 结构持久存在，状态变化才更新，不做每帧重绘。**组件化**，UI 元素是 GameObject 加 Component 的组合，可组合、可复用。**延迟重建**，属性变化只打 Dirty 标记，统一在渲染前调度处理，把一帧内的多次修改合并成一次。**CPU 生成 Mesh**，UI 内容变化时在 CPU 侧重新生成顶点数据，灵活，但这是主要的性能成本，每帧改 Text 这类操作会持续触发重建。**自动合批**，系统按材质、纹理、Stencil 状态合并 DrawCall，合批条件很严格，第 9 章专门讲。

这套取舍和第三方 UI 方案的区别也在这里：不少第三方方案把网格的生成和更新放到 GPU 端，CPU 不参与；UGUI 相反，顶点始终由 CPU 生成。理解这一点，就能解释 UGUI 大多数性能问题为什么集中在 CPU 侧重建。

---

## 1.4 UGUI 的三层架构

UGUI 系统的整体架构可以分成三个逻辑层：

```
┌─────────────────────────────────────────┐
│          组件层（Component Layer）         │
│  Graphic / Image / Text                 │
│  Selectable / Button / LayoutGroup      │
│  职责：UI 结构定义、顶点数据生成、交互逻辑   │
│  产出：调用 CanvasRenderer.SetMesh/SetMaterial │
├─────────────────────────────────────────┤
│          提交层（Submission Layer）        │
│  CanvasRenderer（每个 UI 元素的数据容器）   │
│  Canvas（遍历 CanvasRenderer，调用        │
│         BuildBatch() 合批提交 DrawCall）  │
│  CanvasUpdateRegistry / LayoutRebuilder  │
│  （调度服务，决定组件何时把数据写入）        │
├─────────────────────────────────────────┤
│           执行层（Render Layer）           │
│  Mesh / Material / Texture（GPU 数据）    │
│  Shader（UI/Default）                    │
│  Blend / Stencil / Depth Test（GPU 状态） │
│  职责：执行 DrawCall，完成像素绘制          │
└─────────────────────────────────────────┘
              ↑
        Canvas.BuildBatch()
        （CPU → GPU 的分界线）
```

### 组件层

组件层是开发者直接接触最多的层面。每个 UI 元素由若干 Component 组成：

- **Graphic 系**（继承自 Graphic）：Image、RawImage、Text，负责生成顶点数据
- **交互系**（继承自 Selectable）：Button、Toggle、Slider、Dropdown，负责交互状态
- **布局系**（继承自 LayoutGroup）：LayoutGroup 及其子类，负责子元素排列

### 提交层

提交层保存并提交组件层生成的数据，是数据从 CPU 到 GPU 的通道：

- **CanvasRenderer**：每个 UI 元素对应一个，保存 Mesh、Material、Texture。Graphic 调用 `SetMesh/SetMaterial` 写入数据，它只存不处理
- **Canvas**：渲染入口，每帧渲染前调用引擎内置 native 方法 `Canvas.BuildBatch()`，遍历该 Canvas 下所有 CanvasRenderer 中存储的数据，将同材质同纹理的 Mesh 合并为同一个 DrawCall 提交给 GPU
- **调度服务**：CanvasUpdateRegistry 监听 `Canvas.willRenderCanvases` 事件，统一调度 Layout 重建和 Graphic 重建；LayoutRebuilder 负责布局计算。它们决定组件层何时把数据写入 CanvasRenderer，本身不参与数据

### 渲染层

渲染层是 GPU 侧的执行阶段，接收 Canvas 提交的 DrawCall 并完成像素绘制：

- **Mesh / Material / Texture**：GPU 读入的顶点数据和纹理资源
- **Shader（UI/Default）**：执行顶点变换和片元着色，核心逻辑为 `tex2D × IN.color`
- **Blend / Stencil / Depth Test**：GPU 固定管线状态，控制透明混合和裁剪

EventSystem 不在这条渲染链路上，它属于 1.6 的交互体系，按"输入采集 → 命中检测 → 事件分发"独立运转。

---

## 1.5 UGUI 的渲染本质

这是整个 UGUI 体系中最重要的一句话：

> **所有 UI 最终都转换为 Mesh 数据提交给 GPU。**

在 UGUI 眼中，不存在"按钮"、"图片"、"文字"这些概念，只有顶点、三角形、UV 和颜色。

一个 Image 在 GPU 眼里就是这样的：

```
4 个顶点（四个角）→ 2 个三角形（6 个索引）
→ 一个 Quad（四边形网格）
→ 贴上纹理 → 一个 DrawCall
```

### 从 UI 元素到屏幕像素的完整链路

```
Image/Text 组件
  ↓ OnPopulateMesh(VertexHelper vh)
Mesh（顶点 + 三角形 + UV + 颜色）
  ↓ canvasRenderer.SetMesh(mesh)
CanvasRenderer（存储 Mesh）
  ↓ Canvas.BuildBatch()（引擎 native 方法）
合并 + 提交 DrawCall
  ↓
GPU Vertex Shader → 光栅化 → Fragment Shader → FrameBuffer
```

### 源码视角：Graphic.cs 中的 Rebuild

下面是简化后的 `Graphic.cs` 骨架（完整源码见 uGUI 仓库 `com.unity.ugui/Runtime/UGUI/UI/Core/Graphic.cs`）：

```csharp
public class Graphic : UIBehaviour, ICanvasElement {
    // Dirty 标记
    public virtual void SetVerticesDirty() {
        m_VertsDirty = true;
        CanvasUpdateRegistry.RegisterCanvasElementForGraphicRebuild(this);
    }

    // 重建入口
    public virtual void Rebuild(CanvasUpdate update) {
        if (update == CanvasUpdate.PreRender) {
            UpdateGeometry();   // 阶段一：更新顶点
            UpdateMaterial();   // 阶段二：更新材质
        }
    }

    // 顶点重建
    private void UpdateGeometry() {
        DoMeshGeneration();                  // 调用子类的 OnPopulateMesh 生成顶点
        canvasRenderer.SetMesh(m_WorkerMesh); // 提交给 CanvasRenderer
    }

    // 子类重写此方法生成自己的顶点
    protected virtual void OnPopulateMesh(VertexHelper vh) {
        // Image 重写 → 生成 4 个顶点 + 2 个三角形
        // Text 重写 → 每个字符生成一个 Quad
    }

    // 材质更新
    private void UpdateMaterial() {
        canvasRenderer.SetMaterial(material, mainTexture);
    }
}
```

这一节只说明两件事：顶点由谁生成（Graphic 的子类），提交给谁（CanvasRenderer）。完整的重建调度在第 4 章。

---

## 1.6 UGUI 的系统组成全景

层次划分回答"数据怎么流动"，系统组成回答"UGUI 由哪几块构成"。UGUI 可以分成三大体系加一组支撑系统：

| 体系 | 核心组件 | 负责什么 | 详见章节 |
|------|---------|---------|---------|
| 空间与布局 | RectTransform、LayoutGroup、LayoutRebuilder、ContentSizeFitter | 位置、尺寸、排列 | 第 3、11 章 |
| 渲染 | Graphic 家族、Canvas、CanvasRenderer | 顶点生成、合批、提交 | 第 5、6、8、9 章 |
| 交互 | EventSystem、InputModule、Raycaster、ExecuteEvents、Selectable | 输入采集、命中检测、事件分发 | 第 12、13 章 |
| 资源与图集 | Sprite、SpriteAtlas、动态图集 | 纹理组织与加载 | 第 14 章 |
| 裁剪与文本 | Mask、RectMask2D、Text/TMP、Shader | 裁剪、字形、着色 | 第 15、16、18 章 |
| 性能与工具 | Profiler、Frame Debugger、RenderDoc | 定位与优化 | 第 20、21、24 章 |

三大体系的关系是闭环：交互层把输入变成事件，回调修改布局和渲染数据，渲染层重建并提交，下一帧输入读取新状态。

### 真实场景中的层级结构

把这些系统落到场景里，一个最简单的 UI 界面长这样：

```
Scene
├── EventSystem              ← 交互体系（含 StandaloneInputModule + GraphicRaycaster）
└── Canvas                   ← 渲染入口，所有 UI 必须挂在它下面
    ├── CanvasScaler         ← 分辨率适配（第 7 章）
    ├── Background（Image）
    └── Panel
        ├── Title（Text）
        └── CloseButton（Button）
            ├── Image        ← 显示部分（RectTransform + Graphic + CanvasRenderer）
            └── Button       ← 交互部分（Selectable 状态机）
```

每个可见的 UI 元素都由三部分组成：RectTransform 决定位置尺寸，Graphic 子类生成顶点，CanvasRenderer 保存渲染数据。可交互的元素再加一个 Selectable 子类。Canvas 的三种渲染模式（ScreenSpaceOverlay / ScreenSpaceCamera / WorldSpace）和多 Canvas 拆分策略，留到第 6 章展开。

---

## 1.7 组件继承总览

UGUI 的组件体系围绕几个抽象基类展开，理解继承关系就能猜到大部分组件的职责来源（以 uGUI 仓库 main 分支为准）：

```
UIBehaviour（所有 UI 组件的公共基类）
├── Graphic（抽象）                → 可视化组件，实现 ICanvasElement
│   └── MaskableGraphic（抽象）     → 支持 Mask 裁剪
│       ├── Image / RawImage / Text
├── Selectable                     → 交互组件基类
│   ├── Button / Toggle / Slider / Scrollbar / Dropdown / InputField
├── LayoutGroup（抽象）             → 布局容器
│   ├── HorizontalOrVerticalLayoutGroup
│   │   ├── HorizontalLayoutGroup / VerticalLayoutGroup
│   └── GridLayoutGroup
├── ContentSizeFitter / LayoutElement → 布局辅助组件
├── CanvasScaler                   → 分辨率适配
├── Mask / RectMask2D              → 裁剪
├── ScrollRect                     → 滚动容器
├── EventSystem                    → 事件调度
├── BaseInputModule（抽象）         → 输入模块基类
│   └── StandaloneInputModule
└── BaseRaycaster（抽象）           → 射线检测基类
    ├── GraphicRaycaster / PhysicsRaycaster
    └── Physics2DRaycaster         → 继承 PhysicsRaycaster
```

另有几个重要类不继承 UIBehaviour，单独说明：

- `LayoutRebuilder` 只实现 `ICanvasElement`，是布局重建的执行器
- `CanvasUpdateRegistry`、`ExecuteEvents`、`StencilMaterial` 是静态工具类或单例，负责调度和材质管理

---

## 1.8 UI 修改后，一帧内的流转

架构的最后一个视角是时间。下面这条链路不是每帧都完整执行：每帧固定执行的是输入轮询和渲染前的检查，事件分发、布局、重建、合批都是条件触发。以"UI 被输入修改后"的那一帧为例：

```
EventSystem.Update
  → InputModule.Process()             ← 采集输入（第 12 章）
  → EventSystem.RaycastAll()          ← GraphicRaycaster 命中检测
  → ExecuteEvents 分发事件            ← 触发 OnClick 等回调
  → 回调修改 UI 属性                  ← 打 Dirty 标记，不立即重建
  → Canvas 触发 willRenderCanvases
  → CanvasUpdateRegistry.PerformUpdate()
      ├── Layout Rebuild（阶段 0~2）   ← 布局计算（第 11 章）
      ├── ClipperRegistry.Cull()      ← 裁剪更新（第 15 章）
      └── Graphic Rebuild（阶段 3~4）  ← OnPopulateMesh → SetMesh（第 4 章）
  → Canvas.BuildBatch()               ← 合批提交（第 9 章）
  → GPU：顶点着色 → 光栅化 → 片元着色   ← 像素输出（第 10 章）
```

中间每一步是否执行，取决于前一环是否真的产生了变化：没有输入就不分发事件，没有 Dirty 就不重建，Canvas 数据没变就不重新合批。这就是保留模式省性能的方式。后续章节的展开顺序，基本就是这条链路的顺序。

---

## 1.9 源码结构概览

UGUI 的 C# 源码以 Unity Package 形式发布。当前 main 分支的结构如下（旧版仓库如 2019.1 分支路径为 `UnityEngine.UI/UI/`，结构类似）：

```
com.unity.ugui/（GitHub: Unity-Technologies/uGUI, main 分支）
└── Runtime/UGUI/
    ├── UI/Core/                  ← 核心组件
    │   ├── Graphic.cs / MaskableGraphic.cs / Image.cs / RawImage.cs / Text.cs
    │   ├── Selectable.cs / Button.cs / Toggle.cs / Slider.cs / Scrollbar.cs / Dropdown.cs / InputField.cs
    │   ├── Mask.cs / RectMask2D.cs / StencilMaterial.cs / MaskUtilities.cs
    │   ├── CanvasUpdateRegistry.cs / GraphicRegistry.cs / GraphicRaycaster.cs
    │   ├── ScrollRect.cs / VertexHelper.cs / FontUpdateTracker.cs
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

- `Canvas`、`CanvasRenderer`、`RectTransform` 是 Unity 引擎内置组件，位于 `UnityEngine.CoreModule`，不在 uGUI 仓库中。C# 侧只暴露部分 API，核心方法（如 `Canvas.BuildBatch`）标记 `[NativeMethod]`，由引擎 C++ 实现，看不到源码。
- `InputSystemUIInputModule`（新输入系统）不在 uGUI 仓库中，由 Input System 包提供。

本章涉及的核心文件：

| 文件 | 类 | 职责 |
|------|----|------|
| `Canvas`（引擎内置） | Canvas | 渲染入口，触发 BuildBatch |
| `CanvasRenderer`（引擎内置） | CanvasRenderer | 保存 UI 元素的 Mesh 和 Material |
| `Graphic.cs` | Graphic | 抽象基类，定义 Rebuild 和 Dirty 机制 |
| `CanvasUpdateRegistry.cs` | CanvasUpdateRegistry | 统一调度所有 UI 更新 |

---

## 1.10 本章总结

1. **IMGUI vs UGUI**：IMGUI 是即时模式，每帧重绘；UGUI 是保留模式，状态变化时才更新。
2. **设计特点**：保留模式、组件化、延迟重建、CPU 生成 Mesh、自动合批，这五条是后面所有章节的背景。
3. **三层架构**：组件层（Graphic/Selectable/LayoutGroup）→ 提交层（CanvasRenderer/Canvas/调度服务）→ 执行层（GPU）。EventSystem 属于交互体系，不在渲染链路上。
4. **系统组成**：空间与布局、渲染、交互三大体系闭环，加资源图集、裁剪文本、性能工具等支撑系统。
5. **渲染本质**：所有 UI 最终都是 Mesh + Material + Texture，顶点由 CPU 生成。
6. **动态视角**：一帧内是输入 → 事件 → 布局 → 重建 → 合批 → 渲染，只有 Dirty 的部分才重建。

### 推荐的源码阅读路径

```
打开 Graphic.cs → 依次阅读：
  1. SetVerticesDirty()    ← 理解 Dirty 标记机制
  2. SetMaterialDirty()    ← 同上
  3. Rebuild()             ← 理解重建流程的两个阶段
  4. OnPopulateMesh()      ← 虚方法，子类实现生成顶点
  5. UpdateGeometry()      ← 理解 Mesh 如何提交给 CanvasRenderer
```
